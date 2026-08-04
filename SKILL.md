---
name: db-supervisor
description: Fresh-eyes supervision pass for any database work an AI agent just built (schema, RLS policies, migrations, edge functions touching the database, storage policies). The builder never grades its own homework - a separate agent plays the "Investigate" role, proves who-can-see-what with real queries, and gives an explicit OK before shipping. Use after building or changing anything database-related, or when the user asks for a db check / investigate pass / second opinion on database work.
---

# DB Supervisor

Born from [Supabase Evals](https://supabase.com/evals) (July 2026): coding agents score at or near 100% on Build but drop to 67% on Investigate. The fix is structural, not model-sized: after an agent builds database work, a SEPARATE fresh-context agent investigates it and gives the OK. The builder never reviews itself.

> Status: work in progress. This is a start, not a finished methodology. Issues and PRs welcome.

## What it checks, and what it doesn't

| Checks | Doesn't |
|---|---|
| Access: who can see which data, proven with real queries | Optimize your queries or redesign your schema |
| Performance basics via advisors: unindexed foreign keys, RLS policies re-evaluated per row, obvious traps | Load testing or capacity planning |
| Migration drift and declarative-schema bypasses | Write your migrations for you |
| Outdated packages and auth shortcuts | Full code review of your app |
| Storage and Realtime wiring (when touched) | Design decisions - those stay human |

It's a supervisor, not an optimizer: it verifies the work is safe and honest, flags the performance traps that bite later, and gives an explicit verdict. Deep optimization is its own job.

## Trigger rule (default ON)

After completing any of these in a session, run this skill before calling the work done:

- Schema changes (new tables, columns, types)
- RLS policies (new or modified)
- Migrations (applied or written)
- Edge functions that read or write the database
- Storage bucket policies
- Auth configuration touching data access

If the user explicitly says skip it, skip it. Otherwise supervision is part of the definition of done.

## How to run it

Spawn ONE fresh-context subagent (fresh mind is the whole point). A mid-tier model is fine: supervision is checklist work, not architecture. The prompt must include:

1. The project reference and a SHORT factual list of what was just built (tables/policies/functions touched). Do NOT include the builder's reasoning or justifications - the supervisor forms its own view from the live project.
2. Documented project exceptions, as facts: intentional decisions the project has already made ("table X is public on purpose", "function Y is SECURITY DEFINER by design"). Keep these in a small file in the repo (e.g. `supabase/SUPERVISOR-NOTES.md`) so they persist across runs. The supervisor does NOT re-flag a documented exception; it verifies the exception still does only what it says ("public read intentional" - fine, but write better still be locked). This is the difference from stateless advisors that re-flag your intentional decisions every run: context travels, justification doesn't.
3. The checklist below.

The supervisor uses whatever real access exists: Supabase MCP tools (`execute_sql`, `get_advisors`, `list_migrations`, `get_logs`) or the Supabase CLI. Reading code is not enough; every verdict needs a live check.

## Supervisor checklist (the Investigate role)

1. **Who can see which data - with proof.** For each table touched: query as anon, as an authenticated user, and cross-user/cross-tenant. The verdict per table is explicit: "user A cannot read user B's rows: VERIFIED" or "LEAK". Never accept the policy text as proof; run the query.
2. **Advisors, both lenses.** Run the security advisors AND the performance advisors: unindexed foreign keys, RLS policies re-evaluated per row instead of once, missing primary keys, obvious slow patterns. Report anything new since the build.
3. **Migration reality check.** Migration history vs actual schema. Flag drift, and flag hand-written migrations where a declarative schema was the right call. Evals finding: agents hand-write migrations EVEN in projects that already use declarative schemas, so check for this even when `supabase/schemas/` exists. Caveat: `db diff` does NOT capture RLS policies or DML, so the RLS verification in item 1 happens regardless of declarative setup.
4. **Shortcut scan.** Hand-rolled auth verification with `supabase-js` in edge functions where `@supabase/server` applies, other deprecated or legacy packages, service-role usage where anon + RLS should work, `SECURITY DEFINER` without a documented reason.
5. **Timezone and race basics** where relevant: date handling that shifts days across timezones, async mutations without a lock or disabled state.

### The impersonation recipe (for item 1)

You don't need test logins. Postgres can impersonate roles and JWT claims directly, with zero side effects:

```sql
begin;
set local role authenticated;
select set_config('request.jwt.claims',
  '{"sub":"<user-A-uuid>","role":"authenticated"}', true);
select * from orders;   -- expect: ONLY user A's rows
rollback;
```

Swap `authenticated` for `anon` (and drop the claims) to test the anonymous path. Every access verdict in the report should come from a query like this.

**Supervision is read-only.** The `begin ... rollback` wrapper means it never mutates data. If a write test is unavoidable, use a disposable row you delete afterward, or run it on a database branch.

### Storage and Realtime (when touched)

Storage policies live on `storage.objects`, so the same impersonation trick works:

```sql
begin;
set local role anon;
select name from storage.objects where bucket_id = 'private-bucket';  -- expect: 0 rows
rollback;
```

Add one HTTP check from outside: a private bucket's public URL must not serve the file. `curl -s -o /dev/null -w "%{http_code}" https://<project-ref>.supabase.co/storage/v1/object/public/<bucket>/<path>` should NOT return 200. For per-user folders, impersonate user A and query user B's folder: 0 rows or LEAK.

Realtime has two silent killers behind "subscribed but no events arrive":

```sql
-- 1. is the table in the realtime publication at all?
select tablename from pg_publication_tables where pubname = 'supabase_realtime';

-- 2. updates/deletes need replica identity to carry full payloads
select relreplident from pg_class where oid = 'public.<table>'::regclass;  -- 'f' = full
```

And remember realtime respects RLS: if the impersonation test says a user can't SELECT a row, that user gets no events for it either. One test, two verdicts.

## Live data, not training data

The model's knowledge has a date; Supabase ships monthly. The supervisor NEVER judges from memory:

- **Package and pattern verdicts** (which client library, declarative schema vs migrations, auth patterns): check against CURRENT Supabase docs before flagging. What was best practice at training time may be deprecated today, and vice versa.
- **Routing facts expire.** Any "agent X is best at investigating" claim is a snapshot. Re-fetch the live leaderboard before leaning on it (see next section).
- **Receipts, always.** Every verdict ships its evidence: the exact SQL queries run and their output, the docs pages consulted (with URLs), the advisor findings. A human validates the receipts, not the vibes. A verdict without receipts is an opinion.
- **Upstream freshness.** Once per supervision run, check whether the sources moved: latest commit of [supabase/agent-skills](https://github.com/supabase/agent-skills) vs when your skills were installed, and latest commit of [supabase/evals](https://github.com/supabase/evals). One GitHub API call each (`https://api.github.com/repos/supabase/agent-skills/commits?per_page=1`). If upstream moved, say it in the receipts: "upstream agent-skills updated N days ago, consider re-installing". Staying current is not a chore here, it is the whole premise.

## Route by the live leaderboard

Many people pay for more than one agent subscription. Use that. And you never switch tools to do it: the agent you work in stays the orchestrator and invokes the other lanes from inside the session via their CLIs, passing evidence out and receiving the verdict back.

1. Once per session, fetch current standings from https://supabase.com/evals (Supabase maintains it; scores move). The page is client-rendered; if your fetch tool can't read it, say "standings unverified today" in the receipts and use the last known standings instead of silently guessing.
2. List the agent lanes available in this environment: the current session itself, plus any external agent CLIs installed (`codex`, or whatever you pay for).
3. Pick the investigator = the available lane with the highest CURRENT Investigate score. At launch (July 2026) that was Codex / GPT-5.6 sol at 100% vs 67% for everyone else, but the leaderboard decides, not this file. If the builder and the best investigator are different vendors, even better: route by stage, not by loyalty.
4. When routing to an external lane, the orchestrator gathers the evidence first (RLS test query outputs, advisor findings, migration list vs schema) and hands the bundle to the external agent framed as adversarial investigator: "Here is what was built and the evidence. Try to find what's wrong. Explicit verdict per item."
5. If the fetch fails or only one lane exists, fall back to a fresh subagent in the current environment. Fresh eyes matter more than the exact model.

## Output format

```
DB SUPERVISOR - <project> - <date>
==================================
Built: <one line>

 [OK]   rls: profiles      cross-user read blocked (tested)
 [OK]   advisors           no new findings
 [WARN] migrations         hand-written, declarative preferred
 [FAIL] rls: orders        anon can SELECT (proof: query + rows)

VERDICT: NEEDS ATTENTION (1 fail, 1 warn)  |  or:  VERDICT: OK - ship it
```

Every FAIL comes with the proof query and the suggested fix. The main session applies fixes, then re-runs supervision until VERDICT: OK. The OK is the closing act: no VERDICT, no ship.

If the same item still FAILs after two fix-and-rerun cycles, stop. Escalate to the human with the receipts instead of looping: investigating by guessing is the exact failure mode this skill exists to prevent.

## Cost guardrail

One supervisor pass per build batch, not per table. If the session built 5 things, one subagent checks all 5. Re-runs after fixes only re-check what failed.
