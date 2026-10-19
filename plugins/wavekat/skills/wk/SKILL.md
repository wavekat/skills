---
name: wk
description: Use the WaveKat CLI (`wk`) to place and manage real phone calls, play a spoken message or recording on a call, write and publish call flows (IVR menus, reminders, message taking), set up the WaveKat Voice calling engine, and read WaveKat platform projects, annotations and dataset exports. Use when the user mentions WaveKat, `wk`, making or automating a phone call, a call flow, or WaveKat Voice.
---

# WaveKat CLI (`wk`)

`wk` is WaveKat's command-line client. It places real phone calls through
WaveKat Voice, manages call flows on the WaveKat platform, and reads
platform data. Everything here works with no terminal, so you can drive it
end to end.

## Ground rules

- **Calls are real and may cost money.** Confirm the phone number and what
  will be said with the user before every call, including automated ones.
  Only call people who asked to hear from the user (WaveKat's terms forbid
  unsolicited automated calls and recorded messages).
- **Always pass `--json`** and read stdout. The human tables have ANSI codes
  and an unstable layout; never parse them. Exit status is the source of
  truth; errors are one line on stderr.
- **Don't guess flags.** Run `wk <command> --help`. For the full agent
  contract (every `--json` shape, exit codes, quirks, recipes), run
  `wk agents`. It matches the installed version, so prefer it over memory.
- Never put passwords in arguments. SIP passwords go through
  `--password-file`, `--password-command`, `--password-stdin` or
  `WAVEKAT_SIP_PASSWORD`.

## 1. Make sure `wk` is installed

```sh
command -v wk && wk --version
```

If it's missing, stop and ask the user to install it. Don't download or
install it yourself. Point them to the install page:
https://wavekat.com/docs/cli/getting-started/ (macOS and Linux, one command). When they say
it's done, run `wk --version` again and carry on. If `wk` was installed to
`~/.local/bin` and isn't found, ask them to add that folder to `PATH`.

`wk update --check` / `wk update` keep it current. For calls without the
desktop app, `wk voice install` adds the calling engine (section 3).

## 2. Sign in (needed for flows and platform data)

```sh
wk me --json      # exits non-zero if not signed in
```

If not signed in, use the two-step login. You have no terminal, so `wk login`
prints a link and exits:

```sh
wk login                    # prints a sign-in link
# Show the user the link. They open it on any device, click Authorize,
# and tell you the code shown on the page (XXXX-XXXX).
wk login --code ABCD-EFGH   # finishes sign-in; the code works for 10 minutes
```

A wrong code leaves the sign-in waiting, so retry `--code`. Running `wk login`
again starts over. For CI, set `WK_TOKEN='wk_…'` and run `wk login`.

`wk --help` lists only the commands the account can use. New accounts get
calls and flows. If a command seems missing, check `wk me --json`.

## 3. Phone calls (`wk voice`)

Calls need a running WaveKat Voice engine: either the desktop app with
**Settings → Automation** on, or the engine installed on its own.

```sh
wk voice status --json      # engine reachable? which phone lines (SIP accounts)?
wk voice setup --json       # checklist: "ready", and "next[]" with the command for each step left
```

**No app on this machine** (server, container, agent box): set it up with
no prompts.

```sh
wk login                                   # first, see step 2
wk voice install                           # ~90 MB; installs, starts and signs in the engine
wk voice accounts add --user <user> --domain <sip-domain> \
  --password-file <path> --json            # the user's own SIP line; only use details they gave you
wk voice setup --json                      # repeat until "ready": true
```

If the user has no WaveKat sign-in and wants automated calls, tell them that
installing agrees to https://wavekat.com/terms/ and use
`wk voice install --accept-terms`.

**Place calls:**

```sh
wk voice call +14155550123 --wait --json                     # dial and follow the call
wk voice call +14155550123 --say "Your order is ready." --check   # render and check, no dial
wk voice call +14155550123 --say "Your order is ready." --wait --json
wk voice call +14155550123 --play message.mp3 --wait --json  # WAV/MP3/OGG/FLAC/M4A, up to 2 min
wk voice call +14155550123 --flow-file menu.yaml --wait --json   # a one-off flow
wk voice call +14155550123 --flow <flow-id> --wait --json    # a published flow
wk voice call list --json                                    # calls in progress
wk voice call hangup <call-id>
wk voice call transcript <call-id>
wk voice call history --json
```

- Use only one of `--say`, `--play`, `--flow-file`, `--flow` per call.
- With several lines, pass `--account <id>` from `wk voice status --json`.
- `--wait --json` prints NDJSON: one state change per line, plus
  `flow_started` / `flow_step` / `flow_finished` for automated calls.
- Exit codes for `call --wait`: `0` answered and ended normally, `1` couldn't
  place it (engine unreachable, bad input, bad flow, quota), `2` busy or
  declined, `3` failed or dropped, `4` no answer.
- `--say` uses the monthly text-to-speech allowance. Repeated text is cached.
- Default limits: one automated call per line at a time, 20 per hour.

If `status` says the engine isn't reachable: ask the user to open WaveKat
Voice and turn on Settings → Automation, or run `wk voice service start` on
a machine without the app. The Mac App Store build can't be driven from a
terminal; `wk voice install` fixes that.

## 4. Call flows (`wk flows`)

A call flow is a YAML script for a call: greeting, menu, message, hours,
booking, hang up. See [references/flows.md](references/flows.md) for the
format and a menu example.

```sh
wk flows validate flow.yaml --json          # exit 1 = fix "issues" (each names its node and code)
wk flows create flow.yaml --json            # saves a draft; .id is flow_…
wk flows render <flow-id> --dry-run --json  # how many characters voicing will use
wk flows publish <flow-id> --render --json  # voice text prompts, then publish
wk flows show <flow-id> --yaml > flow.yaml  # pull the saved draft to edit
wk flows update <flow-id> flow.yaml --json  # then publish again
wk flows list --json
wk flows voices --json                      # voices for --voice
```

- Leave `id` out of a new file; `create` assigns it.
- Calls play recorded clips only. Text prompts must be voiced (`render`, or
  `publish --render`) or those steps stay silent.
- Saving drops YAML comments. Put notes in a `note:` field.
- After `render`, pull the draft with `show --yaml` before editing locally.
- Plans cap how many flows can be published (1 on Free) and which steps
  are allowed; `publish` returns `plan_limit` or `plan_blocked` if so.
- Own recordings: `prompt: {audio: ./greeting.wav, transcript: "…"}`.
  Platform flows take WAV only, up to 10 MB and 2 minutes.

## 5. Platform data

For accounts with the matching modules:

```sh
wk projects list --json
wk projects show <id> --json
wk annotations list <project-id> --review-status approved --json
wk exports list <project-id> --json
wk exports create <project-id> --name "…" --json   # blocks until the snapshot is ready
wk exports download <export-id> --out ./snapshot
wk api <path>                                       # GET any platform endpoint as JSON
```

Lists paginate (default 20); use `--page` and stop at `totalPages`.

## When something goes wrong

- `couldn't look up` / `couldn't connect to` / `timed out reaching`: a
  network problem on this machine. Retry once connectivity is back.
- `403` or a missing command: check `wk me --json` for the account's
  `role` and `features`.
- Anything else: `wk agents` and `wk <command> --help` first. Report bugs
  to WaveKat support via https://wavekat.com.
