# skills

Public agent skills (`SKILL.md` format) for Claude Code and other agents that read skills.

| Skill | What it does |
|-------|--------------|
| [vega-os-app](skills/vega-os-app/) | Build, test, release and publish a React Native TV app for Amazon Fire TV on Vega OS, with quality gates for a fast, stable and secure app |

## Install

With the [skills CLI](https://skills.sh):

```bash
npx skills add omerfsen/skills --skill vega-os-app
```

Or copy `skills/<name>/` into your project's `.claude/skills/` (or `~/.claude/skills/` for every project). Pin to a commit or tag rather than following `main`.

## License

MIT. Each skill's README lists its sources and any trademarks it mentions.
