# Known Issues — Clipboard (X11 selections, tmux, OSC 52)

## 1. Clipboard/PRIMARY goes empty after Ctrl-Z on a tool that used xclip/xsel

**Symptom:** copy text via some CLI tool, background that tool (`Ctrl-Z`,
job goes to `STOPPED`), then no paste target sees any content in
`CLIPBOARD` or `PRIMARY` — in st, and equally in plain gnome-terminal +
bash, with or without tmux.

**Root cause:** X11 selections are not a passive store; whichever client
called `XSetSelectionOwner` must stay alive and keep pumping its X event
loop to answer `SelectionRequest`. `xclip`/`xsel` implement "copy" by
forking a small resident process that holds the selection until another
owner supersedes it. That forked holder never calls `setsid()` — it
inherits the **process group** of the job that spawned it. `Ctrl-Z` sends
`SIGTSTP` to the entire foreground process group, so the holder freezes
(`STAT=T`) along with the tool. Any paste then hangs/returns nothing until
the job is resumed (`fg`/`SIGCONT`).

Confirmed live on this machine: `ps aux` showed multiple orphaned
`xclip -selection clipboard/primary` processes in `STAT=T`, days old,
across several ptys — fossils of exactly this pattern.

**Fix applied (this repo + this machine):**
- `~/.dotfiles/.tmux.conf`: tmux copy-mode no longer shells out to `xclip`
  (`copy-command`). See §3 below for why and what replaced it.
- General rule for any future integration: never pipe a copy into
  `xclip`/`xsel` directly from a job that can be `Ctrl-Z`'d. If a
  subprocess-based clipboard tool must be used, detach it first:
  `setsid -f xclip -selection clipboard`.
- Preferred long-term fix: use OSC 52 (terminal-native clipboard escape),
  which puts ownership in the terminal emulator's own X connection —
  never part of any shell job's process group, so `Ctrl-Z` can't touch it.
  st already implements OSC 52 (`st.c`, `strhandle()`, OSC `52` case →
  `xsetsel()` + `xclipcopy()`, sets both `PRIMARY` and `CLIPBOARD`).

## 2. st's OSC 52 buffer silently truncates copies over ~260 bytes

**Symptom:** OSC 52 clipboard sets work for short strings but corrupt or
drop larger ones (e.g. a multi-line tmux copy-mode selection, a pasted
code block).

**Root cause:** `STR_BUF_SIZ` (the raw buffer for OSC/DCS string
sequences) was aliased to `ESC_BUF_SIZ` = `128*UTF_SIZ` = **512 bytes**.
Base64-encoded clipboard payload overhead (~33%) plus the `52;c;` prefix
leaves room for roughly 260 bytes of real plaintext. Once the escape
sequence exceeds the buffer, `st.c` (`tputc`, around the `ESC_STR`
handling) intentionally `return`s without terminating the sequence — a
documented tradeoff in the code ("better than silently failing with
unknown characters... users will report back") — so the terminal just
stops accepting the rest of that escape until a terminator arrives,
producing a truncated/garbage base64 string.

**Fix:** decoupled `STR_BUF_SIZ` from `ESC_BUF_SIZ` and bumped it to 1 MiB
in `st.c`:

```c
#define STR_BUF_SIZ   (1024*1024) /* big enough for OSC 52 (tmux/vim) clipboard payloads */
```

Verified: with the old 512-byte binary, a 2000-byte copy through real
tmux copy-mode landed in the X clipboard truncated to 379 bytes; with the
rebuilt binary, the same copy landed intact at 2001 bytes (payload +
newline).

**Status:** patched and rebuilt at `~/projects/st/st`. **Not yet
installed** to `/usr/local/bin/st` (left alone intentionally while
debugging). To cut over:

```sh
cd ~/projects/st && sudo make install
```

## 3. This machine's tmux build cannot forward OSC 52 natively (bug, not fixed here)

**Symptom:** tmux's own clipboard integration (`set-clipboard on`, native
`Ms`-capability forwarding used by copy-mode and by any app inside a pane
emitting raw OSC 52) never reaches the outer terminal — no error surfaced
to the user, just silently no-op.

**Root cause (confirmed via `tmux -vv` server log):**
`tty_term_string_ss()` in `tty-term.c` calls ncurses `tiparm_s()` to
expand the two-string-parameter `Ms` capability
(`\E]52;%p1%s;%p2%s\007`). It returns `NULL` unconditionally:

```
could not expand Ms
```

This happens regardless of whether `Ms` comes from `terminal-features`
auto-detection or an explicit `terminal-overrides` — the failure is in the
`tiparm_s()`/ncurses interaction itself, not the capability string. Build
in question: `~/.local/bin/tmux` (`tmux next-3.7`), compiled from
`~/projects/tmux`, linked against `/lib/x86_64-linux-gnu/libtinfo.so.6`.
Likely an `HAVE_TIPARM_S`/`HAVE_TIPARM` configure-time detection mismatch
against the system's ncurses ABI. Not investigated further — out of scope
for the clipboard fix; would require patching/rebuilding tmux itself.

**Workaround applied** (`~/.dotfiles/.tmux.conf`):
- `copy-command` points at `~/.local/bin/tmux-osc52-copy`, a script that
  writes a plain OSC 52 sequence directly to the attached client's tty
  (`tmux display-message -p '#{client_tty}'`), bypassing tmux's broken
  `Ms`-expansion path entirely. `copy-pipe`/`copy-pipe-and-cancel` runs
  this as a job forked from the tmux **server** (never in any pane's
  process group), so it's also immune to issue #1 above.
- `allow-passthrough on` — lets pane-internal apps that already DCS-wrap
  their own OSC 52 for tmux (neovim's clipboard provider, `oscyank`, etc.)
  bypass the same broken `Ms` path and reach st directly, without needing
  `copy-command` at all.

Verified end-to-end on the real tmux session (throwaway test window):
real copy-mode select → `copy-pipe-and-cancel` → script → `xclip -o
-selection clipboard` returns the exact selected text.

**If you want the actual tmux bug fixed** (rather than routed around):
start from `tty_term_string_ss()`/`tty_term_string_s()` in
`~/projects/tmux/tty-term.c` and `configure.ac`'s `tiparm`/`tiparm_s`
detection; compare against what `libtinfo.so.6` actually exports.

## 4. vim's `<leader>y` (`<space>y`) shelled out to `xclip`, and stole the next keystroke into the buffer

**Symptom:** `<space>y` in vim's Visual mode (leader mapped to `<Space>` in
`~/.vimrc`) appeared to not copy at all, while a tmux copy-mode `space`+`y`
selection or a plain mouse/st selection copy worked fine. Separately,
whatever was typed right after `<space>y` sometimes ended up inserted
literally into the vim buffer, even in a window the user had already moved
away from.

**Root cause:** the old mapping was
`vnoremap <leader>y <esc>:'<,'>w !xclip -sel clip<cr><cr>` — a `:w !cmd`
filter that pipes the visual range's lines into `xclip`'s stdin. Reproduced
live (spawn vim, select a line, send the mapping's keys):

1. `xclip -sel clip` forks with the **same pgid as vim itself**
   (`ps -o pid,pgid`: both `2062545`) — the exact holder-inherits-process-
   group pattern from Issue #1. A later `Ctrl-Z` on vim SIGTSTPs the holder
   too and the clipboard goes stale/empty until vim is resumed — matching
   "only tmux/st selection works."
2. The `:w !cmd` filter leaves a `Press ENTER or type command to continue`
   prompt. The mapping's trailing `<cr><cr>` is supposed to auto-dismiss it,
   but does not reliably do so (confirmed: the prompt was still showing,
   unchanged, well after the mapping finished running). The **next
   keystroke typed** is consumed to clear that prompt and then replayed as a
   command — e.g. typing `i` clears the prompt *and* opens Insert mode, and
   everything typed next lands in the buffer. Reproduced exactly: after the
   stuck prompt, sending `ihello-injected` left the buffer reading
   `hello-injectedline·one·to·copy` in `-- INSERT --`.

**Fix applied** (`~/.dotfiles/.vimrc`, symlinked from `~/.vimrc`): replaced
the mapping with a `writefile()`-to-`/dev/tty` OSC 52 sequence (base64 via
`system()`, which exits immediately and holds nothing open — unlike
`xclip`/`xsel`), tmux-DCS-wrapped when `$TMUX` is set so it rides the same
`allow-passthrough` path as Issue #3's fix:

```vim
function! s:OscYank(lines) abort
  let l:b64 = substitute(system('base64', join(a:lines, "\n")), "\n", '', 'g')
  let l:osc52 = "\x1b]52;c;" . l:b64 . "\x07"
  if exists('$TMUX')
    let l:osc52 = "\x1bPtmux;\x1b" . substitute(l:osc52, "\x1b", "\x1b\x1b", 'g') . "\x1b\\"
  endif
  call writefile([l:osc52], '/dev/tty', 'b')
endfunction
vnoremap <leader>y <esc>:call <SID>OscYank(getline("'<", "'>"))<cr>
```

No external process is left running, no message is printed, so there is no
hit-enter prompt to steal the next keystroke. Verified: re-ran the same
repro (select line, `<space>y`) — lands back in `NORMAL` immediately, no
prompt, no `xclip`/`base64` process left in `ps`, buffer untouched.

**Also fixed:** `vnoremap <C-v> :w !xclip -sel clip -o<CR><CR>` (paste) had
the identical `:w !cmd` hit-enter-prompt hazard and, being `-o` (output)
mode with piped-in stdin ignored, never actually pasted into the buffer
either way — it only ever *displayed* xclip's stdout as a message. Replaced
with a silent register-based visual paste:

```vim
function! s:XclipPaste() abort
  let @z = system('xclip -o -selection clipboard')
  normal! gv"zp
endfunction
vnoremap <C-v> <esc>:call <SID>XclipPaste()<cr>
```

`xclip -o` is a one-shot `XConvertSelection` + exit — unlike a copy/set call
it forks no resident selection-holder, so issue #1's Ctrl-Z freeze does not
apply to reads. Verified: primed the clipboard, selected a line, `<C-v>`
replaced it with the clipboard text, no prompt, no leftover process.

## 5. st now answers OSC 52 clipboard-*read* queries — but this machine's vim can't consume it yet

Added genuine OSC 52 query support to `st.c`/`x.c` (`\e]52;c;?\a` /
`\e]52;p;?\a`): `strhandle()`'s case 52 now recognizes Pd=`?` and calls the
new `xselpaste(pc)` (`win.h`), which issues the same `XConvertSelection`
`clippaste()`/`selpaste()` already use. `selnotify()` now branches on a new
`osc52query` flag: instead of raw-pasting the fetched selection into the
pty (its normal Shift-Insert/middle-click job), it accumulates the exact
bytes (skipping the `\n`->`\r` rewrite, which would corrupt the payload),
base64-encodes them (new `base64enc()`), and `ttywrite()`s
`\e]52;<pc>;<base64>\e\` back to the pty. This supersedes the old "no query
response" claim above.

**Verified** with a raw-termios Python harness driving the rebuilt `st -e`
directly (plain shell `read`/`dd` under canonical-mode buffering is *not* a
valid test here — no newline ever terminates an OSC reply, so a canonical
tty just withholds it; that cost an hour of chasing a nonexistent bug before
switching to a proper raw-mode reader):
- Primed `CLIPBOARD`/`PRIMARY` via `xclip`, queried both from inside a fresh
  `st`: got back the exact base64 of each, decoded byte-for-byte correct,
  in ~1-130ms.
- Empty clipboard: response is `\e]52;c;\e\` (empty payload), no crash —
  guards an `xrealloc(qbuf, 0)` edge case in the accumulator.
- Clean rebuild (`make`) has no new warnings.

**Installed** to `/usr/local/bin/st` (`sudo make install`, confirmed
byte-identical via `cmp`) — live for real terminal sessions.

**Why this doesn't unblock a vim-side OSC 52 *paste* today:** the natural
consumer is Vim's `TermResponseAll` autocmd + `v:termosc`, which decode an
arbitrary incoming OSC/DCS/APC response. This machine's vim is
`9.1.0016-1ubuntu7.18` (Ubuntu's security-backport track — patch "16" base,
not a real sequential upstream patch count); `TermResponseAll`/`v:termosc`
landed much later upstream and don't exist here:
```
$ vim -es -c 'echo exists("##TermResponseAll")' -c 'q!'
0
```
Sourcing the `TermResponseAll`-based autocmd on this build throws
`E216: No such group or event: TermResponseAll osc {` — adding it to
`.vimrc` as-is would break vimrc on every vim startup, so it was **not**
added. The working paste path stays the `xclip -o` mapping fixed in #4
above. Unblocking the vim-native route needs a vim rebuilt/repackaged past
upstream patch ~9.1.0517 (not attempted — out of scope for this repo).
