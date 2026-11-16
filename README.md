# Next Quarter Skills for Claude

Next Quarter's skills for Claude, covering integrated business planning, supply chain and finance. Each business domain is packaged as its own **plugin** so teams install only what they need.

| Plugin | Domain | Skills |
|---|---|---|
| [`nextq-demand-planning`](plugins/nextq-demand-planning/) | Demand Planning & Forecasting (supply chain, S&OP / IBP, inventory) | 25 |

## Repository layout

```
.claude-plugin/marketplace.json     Lists every plugin in this repo (marketplace name: "nextq")
LICENSE                             Apache-2.0, covers every plugin
plugins/
  <plugin-name>/                    One folder per domain = one plugin = one directory listing
    .claude-plugin/plugin.json      Plugin manifest (name, version, description, author)
    README.md                       What the plugin does and what it runs or sends
    skills/<skill-name>/SKILL.md    One folder per skill
```

## Installing

**Claude Code**
```
claude plugin marketplace add forecastera/nextq-skills
claude plugin install nextq-demand-planning@nextq
```

**Your organization on claude.ai (Team / Enterprise):** an Owner adds this repository under **Organization settings > Plugins & skills > Marketplaces**, then sets the plugin to *Available to install* or *Installed by default*. Skills then work in Claude chat, Cowork and Claude Code.

## Adding skills

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[Apache-2.0](LICENSE)
