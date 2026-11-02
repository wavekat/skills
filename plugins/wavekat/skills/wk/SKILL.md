---
name: wk
description: Use the WaveKat CLI (`wk`) to place and manage real phone calls, play a spoken message or recording on a call, write and publish call flows (IVR menus, reminders, message taking), ask someone a question by phone and read the key they press (including calling the user when you are blocked), set up the WaveKat Voice calling engine, and read WaveKat platform projects, annotations and dataset exports. Use when the user mentions WaveKat, `wk`, making or automating a phone call, a call flow, or WaveKat Voice.
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

## 4. Ask someone a question by phone

Use this when you need an answer from a person and a key press is enough:
"Is the 3 pm appointment still on? Press 1 for yes, 2 for no." The person
called must have asked to hear from the user. One published flow, **Ask a
question**, serves every question. Its variables all have defaults, so it
also plays with none given.

**First use.** The flow file is `references/ask-question.yaml` in this
skill's directory (resolve it from the base directory this skill was loaded
from, not the working directory). Run the first call with `--check --json`
to confirm the engine supports `--var`; if it errors on that flag, tell the
user to run `wk voice update` (do not read exit code 2 there as "busy").
Look for the flow; create it only if it is missing:

```sh
wk flows list --json        # a published "Ask a question"? An unpublished draft of that name: reuse it (publish --render)
# Missing: tell the user and ask before creating it. Say it needs the Pro
# plan (it has a menu) and uses one of their published-flow slots. Then:
wk flows create <this skill's directory>/references/ask-question.yaml --json   # .id is flow_…
wk flows publish <flow-id> --render --json   # voices the defaults, so a call that keeps them is free
```

If the account can't publish (`plan_blocked`, or `canPublish: false` in
`wk flows list --json`) or the user declines, use the same flow without
publishing: replace `--flow "Ask a question"` with
`--flow-file <this skill's directory>/references/ask-question.yaml` and keep
the same `--var` options. Nothing is created or published.

**Each question** is one call (confirm with the user first, as in the Ground rules):

```sh
wk voice call +14155550123 --flow "Ask a question" \
  --var intro="Claude, working on yans dot app, is blocked." \
  --var question="Should I delete the old migration?" \
  --var option1=Yes --var option2=No --wait --json
```

- `--var name=value` can repeat. For text with quotes or `$`, use
  `--vars-file values.json`. `--check` prepares the call without dialing.
  `option3` and `option4` are optional.
- `intro`, `question` and the options are read aloud: write names the way
  they are said ("yans dot app"). Keep it short. Text from `--var` is voiced
  when the call is placed and uses the prompt quota (repeats are cached).
- The `flow_finished` line has `choices: [{node, digit, label, loops}]`. The
  answer is the last choice with `loops: false`. Key 9 (repeat) has
  `loops: true`, so it is never an answer. No such choice, an exit code
  other than 0, or `completed: false` means no answer.

**Phoning your own user when you are blocked.** Only if the user asked for
it in this session and gave you a number. Confirm the number once; that, with
their "call me if you get stuck", covers the calls for this session, to that
number, with questions for that user only. Every other call follows the
Ground rules.

- Ask one question per call, with 2 to 4 short options, and name the project
  in `intro`. Ask only the most blocking question; the rest wait for the user
  or a later call. Never place several calls in a row (20 automated calls
  per hour at most).
- The question goes to text-to-speech and is read on a phone line. Never put
  secrets, tokens or private data in it.
- Act on the option pressed and carry on. Tell the user in the terminal what
  was asked and what they chose by phone, so they can see it when they return.
- No answer: do not guess and do not call again for the same question. Write
  the question in the terminal and wait, or carry on only with work that
  doesn't depend on the answer.

## 5. Call flows (`wk flows`)

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
- Calls play recorded clips only, except text in a flow with variables,
  which is voiced when the call is placed. Other text prompts must be
  voiced (`render`, or `publish --render`) or those steps stay silent.
- Saving drops YAML comments. Put notes in a `note:` field.
- After `render`, pull the draft with `show --yaml` before editing locally.
- Plans cap how many flows can be published (1 on Free) and which steps
  are allowed; `publish` returns `plan_limit` or `plan_blocked` if so.
- Own recordings: `prompt: {audio: ./greeting.wav, transcript: "…"}`.
  Platform flows take WAV only, up to 10 MB and 2 minutes.

## 6. Platform data

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
