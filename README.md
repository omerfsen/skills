# skills

[![skills.sh](https://skills.sh/b/omerfsen/skills)](https://skills.sh/omerfsen/skills)

Public agent skills (`SKILL.md` format) for Claude Code and other agents that read skills.

| Skill | What it does |
|-------|--------------|
| [vega-os-app](skills/vega-os-app/) | Build, test, release and publish a React Native TV app for Amazon Fire TV on Vega OS, with quality gates for a fast, stable and secure app |

## Install

With the [skills CLI](https://skills.sh):

```bash
npx skills add omerfsen/skills --skill vega-os-app
```

As a Claude Code plugin:

```bash
/plugin marketplace add omerfsen/skills
/plugin install vega-os-app@omerfsen-skills
```

Or copy `skills/<name>/` into your project's `.claude/skills/` (or `~/.claude/skills/` for every project). Pin to a release tag (see [Releases](https://github.com/omerfsen/skills/releases)) rather than following `main`.

## License

MIT. Each skill's README lists its sources and any trademarks it mentions.
