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

## Links

- CLI docs: <https://wavekat.com/docs/cli/>
- Call flow format: <https://github.com/wavekat/wavekat-flow>
- Support: <https://wavekat.com>

## License

[MIT](LICENSE). The `wk` binary itself is covered by its own license.
