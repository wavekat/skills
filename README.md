# WaveKat skills for AI agents

Plugins that let Claude (and other agents) use [WaveKat](https://wavekat.com)
through the `wk` command-line tool: place real phone calls, play a message
or recording on a call, and write and publish call flows.

## Install in Claude Code

```sh
claude plugin marketplace add wavekat/skills
claude plugin install wavekat@wavekat-skills
```

Or inside a Claude Code session:

```
/plugin marketplace add wavekat/skills
/plugin install wavekat@wavekat-skills
```

Then ask Claude something like *"call +1 415 555 0123 and tell them their
order is ready"* or *"build a call flow that asks people to press 1 to
confirm"*. If `wk` isn't installed yet, Claude offers to install it.

## What's here

| Plugin | What it does |
|--------|--------------|
| [`wavekat`](plugins/wavekat) | The `wk` skill: install and sign in, phone calls with `wk voice`, call flows with `wk flows`, and WaveKat platform data |

## Without Claude Code

The skill is plain Markdown in
[`plugins/wavekat/skills/wk/SKILL.md`](plugins/wavekat/skills/wk/SKILL.md).
Any agent that runs shell commands can use it. The full agent guide for the
installed version is always one command away:

```sh
curl -fsSL https://dl.wavekat.com/wk/install.sh | sh
wk agents
```

## Data and privacy

The plugin is instructions only: it has no connectors and no code of its own.
Claude runs the `wk` CLI on your machine, and `wk` talks to these services:

| Service | What it receives | When |
|---------|------------------|------|
| WaveKat platform (`platform.wavekat.com`, hosted on Cloudflare) | Your WaveKat sign-in, call flows you create, and (with Sync to cloud on) call history, recordings and transcripts | `wk login`, `wk flows`, platform data commands, and calls from a signed-in engine |
| Your own SIP / phone provider | The numbers you call and the call audio | `wk voice call` |
| ElevenLabs (through WaveKat) | The text of prompts being turned into speech | `wk voice call --say`, `wk flows render` / `publish --render` |
| Sentry | Anonymous crash and error reports, with personal fields removed | When a `wk` command fails; turn off with `wk config telemetry off` |

What is kept and for how long is set out in the
[WaveKat privacy policy](https://wavekat.com/privacy/): call data synced to
a WaveKat account is kept until you delete it. Recordings and transcripts
from calls stay on your computer unless you sign in with Sync to cloud on.
Use is covered by the [WaveKat terms](https://wavekat.com/terms/).

## Icon

`plugins/wavekat/.claude-plugin/icon.png` is generated from
[wavekat-brand](https://github.com/wavekat/wavekat-brand)
(`assets/logos/wavekat-icon-light.svg`). Don't edit it here: change the
brand SVG, then run `scripts/sync-icon.sh` or the **Sync brand icon**
workflow (it also runs daily). `scripts/icon.lock` records which brand
commit the PNG came from.

## Links

- CLI docs: <https://wavekat.com/docs/cli/>
- Call flow format: <https://github.com/wavekat/wavekat-flow>
- Support: <https://wavekat.com>

## License

[MIT](LICENSE). The `wk` binary itself is covered by its own license.
