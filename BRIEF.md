# MinkaTrace — handover brief

## What this is

A MinkaDE-specific forensics tool. **Given a moment in time, assemble everything that was
happening.** The compositor log with its timezone already corrected, the journal, coredumps,
what binary each relevant process was actually executing, and live IPC state if the session is
still up.

It is not a crash reporter, a log viewer, or a monitor. MinkaMon watches the system live;
MinkaTrace reconstructs what happened after it is over.

## The governing principle: assemble, do not conclude

Every serious debugging mistake in this project came from a confident wrong reading of
incomplete evidence, not from missing evidence:

- Log silence was read as absence when the code never logged that path at all.
- UTC timestamps were compared against local ones, producing hours of phantom timeline.
- A buffer type was inferred from a compositor scanout sample that did not mean what it looked
  like it meant.

So the output must state what each source **cannot** tell you, not only what it says. A line
like

> compositor log covers 13:20–13:35Z at INFO and above — focus changes are `debug!` and will
> not appear here

is worth more than any inference the tool could make. Present the evidence and its limits;
let the reader draw the conclusion. When a source is empty, say whether it is *empty* or
*disabled* — those are different facts and the difference has cost real time.

## Evidence sources

Each entry below is a verified property of this machine, not a guess. Encode the traps.

### 1. Compositor session logs — `~/shoji_wm/logs`

- **Timestamps inside the files are UTC.** Everything else on this system is local (BST in
  summer). This is the single highest-value correction the tool makes.
- **Level is INFO and above.** No DEBUG, no TRACE. Whole categories of event — focus changes
  among them — are logged at `debug!` and are therefore *structurally absent*, not missing.
- **`latest.log` is the live session.** It is being appended to right now.
- **Numbered filenames are rotation times, not session starts.** `<epoch_ms>.log` decodes to
  the moment that file was closed — which is when the *next* session began. Example: file
  `1786982422871.log` decodes to 17/8 17:00:22 local, and the *following* file's content
  begins at 17/8 16:00:22Z, the same instant. Using the filename as a start boundary puts
  every lookup one session early.
- **Therefore: do not parse filenames to find the right log.** Read the first and last
  timestamp inside each file. That is unambiguous.
- The directory is unpruned and has reached tens of gigabytes in the past. Do not read whole
  files into memory; seek.

### 2. systemd journal — `journalctl`

- Local time. Correlating it against source 1 without converting is the classic failure.
- `journalctl --user` for session-scoped units.

### 3. Coredumps — `coredumpctl`

- Local time. `Timestamp:` in `coredumpctl info` is whole seconds; the core filename carries
  microseconds but is often zero-padded, so do not expect sub-second ordering from it.
- **Retention is roughly 12 days** on this machine (observed: 13 entries, oldest 9/8 as of
  21/8). Nothing is configured in `/etc/systemd/coredump.conf` — these are systemd defaults.
  So "only one crash listed" means "none in the last fortnight", never "never".
- `si_code: SI_TKILL` means the process signalled *itself* — typically a crash handler
  re-raising after the real fault. The interesting frame is further up the stack, not frame 0.

### 4. Application crash reporters — check whether they are even on

`~/.mozilla/firefox/Crash Reports/` has written nothing since January 2026. An empty crash
report directory here means **the reporter is disabled**, so coredumpctl is the only record.
Reporting "no crash reports found" without that distinction is actively misleading.

### 5. Which binary is actually running

The running process is frequently *not* the file on disk. Check all three:

```sh
ls -l /usr/bin/shoji_wm                    # what is installed
ps -o lstart= -p "$(pgrep -x shoji_wm)"    # when the running process started
readlink /proc/<pid>/exe                   # trailing " (deleted)" = replaced under it
```

A trailing `(deleted)` means a rebuild replaced the file and the process still holds the old
inode. This bit us on 21/8: the fix was installed at 01:32 and the session was still executing
the image from three days earlier. The same applies to `bartizan-lsp`, which the editor
launches straight out of `target/release/` with no copy on `PATH`.

### 6. Live compositor state — the IPC socket

`$XDG_RUNTIME_DIR/shojiwm-$WAYLAND_DISPLAY.sock`, NDJSON over a Unix socket.
`{"id":1,"method":"workspaces.get"}` returns every window with `focused`, `maximized`,
`minimized` and a `lastFocusedAt` stamp. This is the only way to observe focus from outside,
precisely because focus is logged at `debug!`.

Connect, send, read one line, close. The server drops a half-closed connection, so a bare
`socat` often returns nothing. There is no `windows.list`; methods are registered in
`ShojiWM/packages/config/src/index.tsx` under `WORKSPACE_IPC.handle`.

## Configuration

**No hardcoded paths.** Every location above is a *documented default*, not a literal to
scatter through the code. Resolve each one through a single config layer, in this order:

1. an explicit CLI flag,
2. an environment variable,
3. the default recorded in this document.

MinkaDE already has this convention: `shojiwm-env.fish` (symlinked into fish `conf.d`) exports
`SHOJI_CONFIG`, `MINKA_SHELL_DIR`, `MINKA_SHOT_DIR` and `MINKA_FX_BIN`. Follow the same
`MINKA_*` naming so MinkaTrace's settings sit alongside the rest rather than inventing a
parallel scheme.

Some of these must be derived at runtime regardless and must never be frozen into a constant:

- the IPC socket is `$XDG_RUNTIME_DIR/shojiwm-$WAYLAND_DISPLAY.sock` — both parts vary per
  session;
- the installed compositor path should be resolved, not assumed, since the point of source 5
  is that the running image and the on-disk file disagree;
- the log directory's *contents* change per session, so discover files by reading their
  timestamps rather than by name (see source 1).

What stays fixed is the **knowledge**, not the strings: that one clock is UTC and the others
are local, that focus never appears at INFO, that an empty crash-report directory means the
reporter is off. That is the part worth writing down, and it is why this is a MinkaDE tool
rather than a generic one.

## The core operation

```
minkatrace <when>          # a timestamp, or a pid, or "last crash"
```

...produces one timeline, in a single stated timezone, merging every available source, with an
explicit coverage statement per source: what window it spans, what level of detail it holds,
and what it structurally cannot contain.

## Later, optional: per-client crash extraction

Write this **last**. It is a plugin layer, not the core, and the core is useful without it.

For a stripped binary whose symbols are unavailable, the crash reason can often still be
recovered as a plain string from the core:

```sh
zstdcat "$core" | strings -n 8 | grep -aiE 'MozCrashReason|MOZ_CRASH|protocol error|\[GFX'
```

That single step is what identified a `wp_viewport` protocol error on 21/8 after symbolication
had failed completely. Note the privacy consequence and surface it in the UI: a browser core
contains the browsing session in plain text. Page content and analytics payloads came out of
that same grep. Never upload one anywhere, and warn if the user is about to.

Breakpad debug id from a GNU build id, if a symbol server is ever worth trying: take the first
16 bytes, re-read as a little-endian GUID (u32, u16, u16, then 8 bytes verbatim), append `0`
for the age.

## Constraints

- **Read-only with respect to the live session.** This tool observes; it must never restart,
  kill or reconfigure anything.
- **Never `pkill -f`.** If a process must ever be matched, match exactly. `pkill -f "qs -p ."`
  has already killed a live shell here — the `.` is a regex wildcard.
- **Do not patch ShojiWM.** It is bea4dev's project, and there is an open upstream PR. Read
  its logs and its IPC; change nothing in it.
- Quickshell live-reloads on every file save, so a broken intermediate save in any Quickshell
  app wedges the running session. That is not this tool's directory, but know it before
  touching one.
- The repo root `CLAUDE.md` carries the rest of the project's conventions. Read it first.

## Non-goals

- Not a system monitor — that is MinkaMon.
- Not a general-purpose log tool. It knows this stack — ShojiWM, Quickshell, systemd-coredump.
  Making it compositor-agnostic is not in scope; making its *paths* configurable is (see
  Configuration).
- Not a diagnosis engine. It does not decide what went wrong.
