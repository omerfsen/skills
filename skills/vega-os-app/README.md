# vega-os-app: an agent skill for Amazon Vega OS TV apps

An agent skill (Claude Code and other `SKILL.md`-aware agents) for building, testing, releasing and publishing a React Native TV app on **Amazon Fire TV devices running Vega OS**: Fire TV Stick 4K (3rd gen), 4K Select and HD.

It covers the whole lifecycle, from an empty Mac to the Amazon Appstore, with quality gates for a fast, stable and secure app. Vega facts come from Amazon's docs (Vega **2.0**, checked 2026-11-20) and the fact sheets cite every page; general engineering advice is labelled as such.

## What's inside

```
SKILL.md                      entry point: lifecycle, routing, hard rules
references/
  setup.md                    dev machine, SDK, telemetry, VVD, Fire TV Stick + Developer Mode
  repo-and-app.md             repo bootstrap, templates, manifest.toml, compatibility, monorepo
  dev-loop.md                 build, install, run, Fast Refresh, debug, logs, crashes
  quality-gates.md            KPI targets, memory/lifecycle limits, security, TV UX
  testing-ci.md               Jest with Vega mocks, Appium, remote simulation, CI
  release-publish.md          versioning, signing, Appstore assets, review, updates
  ecosystem.md                official samples, help, Amazon's ADBT MCP, community skills
  agent-prompt.md             a kickoff prompt for a new app
  docs/                       sourced fact sheets (CLI, manifest, performance, media/IAP, security/Appstore, testing/UX)
```

## Install

Copy the `vega-os-app` folder into your project's `.claude/skills/` (or `~/.claude/skills/` for all projects). Then put your project values (package ID, publisher, test device) in the project's `CLAUDE.md`; the skill reads them from there.

## How it relates to Amazon's tools

Amazon's [Builder Tools (ADBT)](https://developer.amazon.com/docs/adbt/get-started) MCP server gives agents live Vega docs and diagnosis. This skill adds the end-to-end workflow, quality gates and release discipline, and works without ADBT.

## Keeping it current

Vega changes between SDK versions. The skill tells the agent to check `vega <command> --help` and the docs before trusting a command, and to prefer the docs when they disagree. Fact sheets list their sources and open questions; re-run the research when a new SDK ships.

Not affiliated with or endorsed by Amazon. "Amazon", "Fire TV" and "Vega" are trademarks of Amazon.com, Inc. or its affiliates.
