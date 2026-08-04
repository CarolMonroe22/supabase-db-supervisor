# supabase-db-supervisor

An agent skill that adds a fresh-eyes supervision pass to any database work an AI agent builds. The builder never grades its own homework.

> **Status: work in progress.** This is v0.1, a starting point published in the open. The checklist, routing logic, and output format will evolve. Issues and PRs welcome.

## Why

[Supabase Evals](https://supabase.com/evals) (open sourced July 2026) measured how well coding agents build real Supabase backends across four stages: Build, Deploy, Investigate, Resolve. The headline: agents score at or near 100% on Build, and most drop to 67% on Investigate, the "something is wrong, figure out why" stage.

So this skill moves the Investigate stage inside the build loop. After an agent builds anything database-related, a separate fresh-context agent investigates it: proves who can see which data with real queries, checks advisors, verifies migration history against reality, and gives an explicit OK before anything ships.

Three principles:

1. **Fresh eyes.** The supervisor is a separate agent that never sees the builder's reasoning, only the live project.
2. **Live data, not training data.** The supervisor consults current Supabase docs and the live Evals leaderboard at run time. A model's knowledge has a date; the ecosystem doesn't.
3. **Receipts.** Every verdict ships its evidence (queries + output, docs consulted, advisor findings) so a human can validate. A verdict without receipts is an opinion.

Bonus: if you pay for more than one agent subscription (many of us do), the skill routes the investigator role to whichever agent currently scores highest on Investigate in the live leaderboard. Route by stage, not by loyalty.

## Install

Download or clone this repo, then copy the skill into your Claude Code skills directory:

```
mkdir -p ~/.claude/skills/db-supervisor
cp SKILL.md ~/.claude/skills/db-supervisor/SKILL.md
```

Works best with the [Supabase MCP server](https://supabase.com/docs/guides/getting-started/mcp) connected, so the supervisor can run real queries, advisors, and migration checks. Pairs well with the official [Supabase agent skills](https://github.com/supabase/agent-skills).

## Credits

Benchmark data and stage definitions from [supabase/evals](https://github.com/supabase/evals) (Apache-2.0).

## License

MIT
