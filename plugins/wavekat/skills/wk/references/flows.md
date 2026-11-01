# Call flow format

A call flow is a YAML document. The schema is public:
https://github.com/wavekat/wavekat-flow (JSON Schema plus valid and invalid
examples). Always check a file with `wk flows validate <file> --json`
before saving or calling; it is the authority, not this page.

## Shape

```yaml
schema_version: 1   # 4 if the flow uses variables
name: Appointment reminder
entry: hello
note: Optional free text. Unlike YAML comments, it is kept when the flow is saved.
nodes:
  hello:
    kind: greeting
    prompt: Hi, this is a reminder about your appointment tomorrow at ten.
    exits: { next: bye }
  bye:
    kind: hangup
    prompt: Thanks, goodbye.
```

`entry` is the first step. Leave `id` out of a new file; `create` assigns
it. Each node has a `kind` and `exits` that name the next node. Every node must
be reachable from `entry`, and every exit must point at a node that exists.

## Step kinds

| kind | What it does | Main fields | Exits |
|------|--------------|-------------|-------|
| `greeting` | plays a prompt | `prompt` | `next` |
| `menu` | plays a prompt, waits for a key press | `prompt`, `options` (`{"1": Label}`), `retries`, `timeout_secs` | one per option key, `no_input`, `invalid` |
| `message` | takes a voicemail | `prompt`, `max_secs`, `tone` | optional |
| `hours` | branches on opening hours | `timezone`, `schedule`, `exceptions` | `open`, `closed` |
| `ring` | rings the user's phone | `timeout_secs` | `no_answer` |
| `transfer` | forwards the call | `target` | optional |
| `hangup` | optional goodbye, then ends | `prompt` | none |
| `book` | books an appointment (`schema_version: 2`) | see the schema | see the schema |

Outbound calls (`wk voice call --flow` / `--flow-file`) can use `greeting`,
`menu`, `message`, `hangup` and `hours`.

## Prompts

A prompt is text or a recording:

```yaml
prompt: Thanks for calling.                                  # text, voiced by `render`
prompt: { audio: ./intro.wav, transcript: "Thanks for calling." }   # own WAV, uploaded on create/update
prompt: { audio: vprompt_abc123, transcript: "Thanks for calling." } # an already uploaded clip
```

Calls only play recordings. Run `wk flows render <id>` (or
`wk flows publish <id> --render`) to voice text prompts.

## Variables (`schema_version: 4`)

A v4 flow can declare `variables` and use `{{name}}` in string prompts, in
the `template` of an audio prompt, and in menu option labels. Names are
`[a-z][a-z0-9_]{0,31}`. Each variable needs a `default` (may be `""`); it
may also have `description` and `max_len` (default 200, at most 500,
counted in characters). `{{#name}}...{{/name}}` keeps its text only when
the value is not empty (no nesting, no else).

```yaml
schema_version: 4
variables:
  place: { description: Where to meet, default: "the front desk", max_len: 80 }
prompt: "Please meet us at {{place}}."
prompt: { audio: vprompt_abc123, template: "Please meet us at {{place}}." }
```

`render` voices prompts with the defaults, so a call that keeps them costs
nothing. Pass values with `wk voice call --flow <id-or-name> --var place="Room 4"`
(or `--vars-file`); they are voiced when the call is placed. Values are
trimmed and never expanded. In v1 to v3 flows `{{` is ordinary text. For a
ready-made question flow, see `ask-question.yaml` and the section
"Ask someone a question by phone" in SKILL.md.

## Example: reminder with a confirm menu

```yaml
schema_version: 1
name: Appointment confirm
entry: ask
nodes:
  ask:
    kind: menu
    prompt: >-
      Hi, this is a reminder about your appointment tomorrow at ten.
      Press 1 to confirm, or 2 if you need to reschedule.
    options: { "1": Confirm, "2": Reschedule }
    retries: 1
    exits: { "1": thanks, "2": callback, no_input: bye, invalid: bye }
  thanks:
    kind: hangup
    prompt: Thanks, see you tomorrow.
  callback:
    kind: message
    prompt: Please leave a good time to call you back after the tone.
  bye:
    kind: hangup
    prompt: We'll try again later. Goodbye.
```

Try it as a one-off with
`wk voice call <number> --flow-file confirm.yaml --wait --json`, after the
user confirms the number. The `flow_step` lines show which key was pressed.

## Example: after-hours inbound flow

```yaml
schema_version: 1
name: After hours
entry: welcome
nodes:
  welcome:
    kind: greeting
    prompt: Thanks for calling!
    exits: { next: check_hours }
  check_hours:
    kind: hours
    timezone: America/New_York
    schedule:
      mon: [{ open: "09:00", close: "17:00" }]
      tue: [{ open: "09:00", close: "17:00" }]
    exceptions:
      - date: 2026-12-25
        closed: true
    exits: { open: front_desk, closed: take_message }
  front_desk:
    kind: ring
    timeout_secs: 25
    exits: { no_answer: take_message }
  take_message:
    kind: message
    prompt: Please leave your name and number after the tone.
```
