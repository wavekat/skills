---
name: wk
description: Use the WaveKat CLI (`wk`) to place real phone calls through the user's own SIP line with no WaveKat account, play a spoken message or recording on a call, ask someone a question by phone and read the key they press (including calling the user when you are blocked or need an important confirmation), write and publish call flows (IVR menus, reminders, message taking), set up the WaveKat Voice calling engine, and read WaveKat platform projects, annotations and dataset exports. Use when the user mentions WaveKat, `wk`, making or automating a phone call, calling them when you are stuck, a call flow, or WaveKat Voice.
---

# WaveKat CLI (`wk`)

`wk` is WaveKat's command-line client. It places real phone calls through
the user's own SIP line, the way curl makes an HTTP request, and needs no
WaveKat account to do it. Signing in is optional and adds better voices,
synced lines and published flows. Everything here works with no terminal,
so you can drive it end to end.

## Ground rules

- **Calls are real and may cost money.** Confirm the phone number and what
  will be said with the user before every call, including automated ones.
  The exceptions are in section 4. Only call
  people who asked to hear from the user (WaveKat's terms forbid unsolicited
  automated calls and recorded messages).
- **Always pass `--json`** and read stdout. The human tables have ANSI codes
  and an unstable layout; never parse them. Exit status is the source of
  truth; errors are one line on stderr.
- **Don't guess flags.** Run `wk <command> --help`. For the full agent
  contract (every `--json` shape, exit codes, quirks, recipes), run
  `wk agents`. It matches the installed version, so prefer it over memory.
- Never put passwords or API keys in arguments, and never say them on a
  call. SIP passwords go through `WK_SIP_PASSWORD_COMMAND`,
  `WK_SIP_PASSWORD_FILE` or `WK_SIP_PASSWORD` for a call. To save a line,
  see `wk voice accounts add --help`.

## 1. Make sure `wk` is installed and current

```sh
command -v wk && wk --version
```

If it's missing, stop and ask the user to install it. Don't download or
install it yourself. Point them to the install page:
https://wavekat.com/docs/cli/getting-started/ (macOS and Linux, one command). When they say
it's done, run `wk --version` again and carry on. If `wk` was installed to
`~/.local/bin` and isn't found, ask them to add that folder to `PATH`.

`wk call` and `wk ask` (sections 3 and 4) need the latest `wk` and calling
engine. If `wk call` or `wk ask` isn't a known command, ask the user to run
`wk update`; if `wk ask` says the engine is too old, ask them to run
`wk voice update`, or use the `--flow-file` recipe in section 4 meanwhile.

## 2. Set up a line (no WaveKat account needed)

A call needs the user's own SIP line. Give it to `wk` through the
environment; it is used for that call only and is never saved or synced:

```sh
export WK_SIP_URI='sip:alice@sip.example.com'      # sip:user@domain[:port][;transport=tcp|udp|tls]
export WK_SIP_PASSWORD_COMMAND='op read op://Private/SIP/password'
# Optional: WK_SIP_AUTH_USER (if it differs from the user), WK_SIP_SERVER (outbound proxy)
```

`export` may not last across separate shell calls in your harness. Set the
variables in the user's shell profile, or put them on the same command line
as the call: `WK_SIP_URI=… WK_SIP_PASSWORD_COMMAND=… wk call …`.

The password comes from the first of `WK_SIP_PASSWORD_COMMAND` (a command
that prints it), `WK_SIP_PASSWORD_FILE` (a file holding it) or
`WK_SIP_PASSWORD`. Prefer the command or the file. Only use line details the
user gave you, and ask them to set these variables rather than typing the
password into the conversation. If a line is already saved on this machine
(`wk voice status --json`), no variables are needed.

Optional: **sign in** for better voices, lines synced across machines and
published flows. Skip it for plain calls.

```sh
wk me --json      # exits non-zero if not signed in
wk login          # prints a sign-in link; show it to the user
# They open it on any device, click Authorize, and tell you the code (XXXX-XXXX).
wk login --code ABCD-EFGH   # finishes sign-in; the code works for 10 minutes
```

A wrong code leaves the sign-in waiting, so retry `--code`. Running `wk login`
again starts over. For CI, set `WK_TOKEN='wk_…'` and run `wk login`.
`wk flows` and platform data (sections 5 and 6) need a sign-in. If a command
seems missing, check `wk me --json`.

## 3. Phone calls (`wk call`)

```sh
wk call +14155550123 --say "The build is green." --json
```

- `wk call <to>` waits for the outcome by default; `--no-wait` returns once
  the call is placed.
- **No engine running is fine.** The call starts the engine itself and stops
  it afterwards. If the engine isn't installed, `wk` asks to download it
  (~90 MB). With no terminal to answer, pass `--yes` so `wk` may download it,
  or ask the user to run `wk voice install`.
- **Terms.** The first call on a machine needs `--accept-terms`, which
  agrees to https://wavekat.com/terms/. Pass it only after the user has
  agreed to the terms.
- **Voice.** `--say` text is voiced by the first available source, in this
  order: `--tts <name>` (an error if it can't be used, never a silent
  fallback), WaveKat when signed in, `OPENAI_API_KEY`, `ELEVENLABS_API_KEY`,
  `--tts-command` / `WK_TTS_COMMAND` (a command that reads the text and
  writes audio), then the computer's own voice. The `--json` result says
  which source voiced the call (`speech`); the system voice is robotic, so
  mention that to the user if it matters.
- Provider keys come from the environment, never the command line (the
  TTS command holds no secret).

```sh
wk call +14155550123 --say "Your order is ready." --check --json   # check, no dial
wk call +14155550123 --play message.mp3 --json   # WAV/MP3/OGG/FLAC/M4A, up to 2 min
wk call +14155550123 --flow-file menu.yaml --json   # a one-off flow
wk call +14155550123 --flow <flow-id> --json     # a published flow (needs sign-in)
wk voice call list --json                        # calls in progress
wk voice call hangup <call-id>
wk voice call transcript <call-id>
wk voice call history --json
```

- Use only one of `--say`, `--play`, `--flow-file`, `--flow` per call.
- With several saved lines, pass `--account <id>` from `wk voice status --json`.
- `--json` prints NDJSON: one state change per line, plus
  `flow_started` / `flow_step` / `flow_finished` for automated calls.
- Exit codes: `0` answered and ended normally, `1` couldn't place it (engine
  unreachable, bad input, bad flow, line wouldn't register, no terms,
  quota), `2` busy or declined, `3` failed or dropped, `4` no answer.
- Default limits: one automated call per line at a time, 20 per hour. A
  second call started while a one-off engine is busy fails with exit 1; wait
  for the first to end.

If `wk` says the engine is running but not reachable, ask the user to turn
on Settings → Automation in WaveKat Voice (or quit it), or run
`wk voice service start` on a machine without the app. A version mismatch
is fixed by `wk voice update`. The Mac App Store build can't be driven from
a terminal; `wk voice install` fixes that. `wk voice status --json` and
`wk voice setup --json` show the engine and saved lines.

## 4. Ask someone a question by phone

Use this when you need an answer from a person and a key press is enough:
"Is the 3 pm appointment still on? Press 1 for yes, 2 for no." The person
called must have asked to hear from the user.

```sh
wk ask +14155550123 "Should I delete the old migration?" \
  --option Yes --option No \
  --intro "Claude, working on yans dot app, is blocked." --json
```

- 2 to 4 `--option` labels map to keys 1 to 4; key 9 repeats the question.
  `--intro` says who is asking. Omit `<to>` to call the user's own number
  (`me.phone`, below). `--tts`, `--account` and `--accept-terms` work as for
  `wk call`. Confirm with the user first, as in the Ground rules (or use the
  standing permission below).
- It phones, reads the question and waits. No sign-in or published flow is
  needed. The `--json` result is
  `{answered, digit, label, call_id, outcome, speech}`: the answer is `label`
  when `answered` is true.
- Exit codes: `0` a valid option was pressed, `1` couldn't place the call,
  `2` busy or declined, `3` failed or dropped, `4` no answer, `5` answered
  but pressed no valid option. Anything but `0` means no answer.
- The question and options are read aloud: write names the way they are
  said ("yans dot app") and keep it short.

**Phoning your own user when you are blocked.** The user stores their own
number once:

```sh
wk config set me.phone +14155550123
wk config get me.phone
```

You have **standing permission** to call that number, with questions for
that user, when both are true: `wk config get me.phone` returns a number,
and the user's instructions (CLAUDE.md or similar) tell you to call them
when you are blocked or for important confirmations. That holds across
sessions, with no need to confirm each call. Without the instruction, or if
the user asked for it only in this session, confirm the number once; that
covers this session, to that number only. Every other call follows the
Ground rules.

- Ask one question per call, with 2 to 4 short options, and name the project
  in `intro`. Ask only the most blocking question; the rest wait for the user
  or a later call. Never place several calls in a row (20 automated calls
  per hour at most). Use a call for things that matter, not small
  decisions.
- The question is read aloud on a phone line, by a voice service when the
  voice source is not local. Never put secrets, tokens or private data in it.
- Act on the option pressed and carry on. Tell the user in the terminal what
  was asked and what they chose by phone, so they can see it when they return.
- No answer (exit `4` or `5`, or `answered: false`): do not guess and do not
  call again for the same question. Write the question in the terminal and
  wait, or carry on only with work that doesn't depend on the answer.

**Older engine.** If `wk ask` says the engine is too old and the user can't
update right now, use the flow in `references/ask-question.yaml` in this
skill's directory (resolve it from the base directory this skill was loaded
from, not the working directory), with `--check --json` first to confirm the
engine supports `--var`:

```sh
wk call +14155550123 --flow-file <this skill's directory>/references/ask-question.yaml \
  --var intro="Claude, working on yans dot app, is blocked." \
  --var question="Should I delete the old migration?" \
  --var option1=Yes --var option2=No --json
```

`--var name=value` can repeat (`option3` and `option4` are optional); for
text with quotes or `$`, use `--vars-file values.json`. The `flow_finished`
line has `choices: [{node, digit, label, loops}]`. The answer is the last
choice with `loops: false`; key 9 has `loops: true`, so it is never an
answer. No such choice, an exit code other than 0, or `completed: false`
means no answer.

**Signed-in option: publish the flow.** Users who want the platform voices
cached can publish it once, with the same file. Look first; create it only
if it is missing, and ask the user before creating it (it needs the Pro plan
because it has a menu, and uses one of their published-flow slots):

```sh
wk flows list --json        # a published "Ask a question"? An unpublished draft of that name: reuse it (publish --render)
wk flows create <this skill's directory>/references/ask-question.yaml --json   # .id is flow_…
wk flows publish <flow-id> --render --json   # voices the defaults, so a call that keeps them is free
wk call +14155550123 --flow "Ask a question" \
  --var question="Should I delete the old migration?" \
  --var option1=Yes --var option2=No --json
```

If the account can't publish (`plan_blocked`, or `canPublish: false` in
`wk flows list --json`), use `wk ask` instead.

## 5. Call flows (`wk flows`, needs sign-in)

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
