# Investigation Studio: a guarded web UI for an AI investigation agent

**Node.js (zero dependencies) · Claude Code headless / Agent skills · MCP · SQL · streaming UI**

## The problem
I had built an AI agent workflow (a Claude Code skill) that investigates fraud cases.
It queries the data warehouse, searches internal knowledge and drafts an investigation
memo. But it ran in a terminal, which suited few investigators, and an agent with
database and document-write access needed firm, visible guardrails.

## My role
Self-initiated and built solo. I designed the agent workflow, the investigation rules
and the guardrail model, and built the Studio with AI-assisted development (Claude Code).

## What I built
A local web app that wraps the agent instead of reimplementing it. Each turn launches
the headless agent CLI with streamed JSON output and session resume, so the skill's
rules, query playbook and data-access hooks stay the single source of truth.

- **Light / Heavy modes:** a quick read of the report only (no database access) for
  triage, or a full investigation with queries and a memo
- **Live activity panel:** every SQL query and its result, every search and file read,
  streamed as it happens, so investigators can audit the agent's reasoning
- **Human-in-the-loop approval:** final documents are created only after the
  investigator reviews the memo, enters the outcome code and ticks "reviewed"
- **Background jobs:** send a long investigation to a dock and keep working;
  you get a notice when it finishes

## Engineering highlights
- **Least privilege per turn.** The server sets a fixed tool allowlist for each turn
  type. Light turns get no database access. Document-write tools exist *only* on the
  approval turn, and the server refuses that turn unless the conclusion qualifies.
  Anything else is denied automatically, so the agent never stalls on a prompt.
  Database access is read-only and enforced by a hook.
- **Local-only security.** Binds to 127.0.0.1, checks the Host header (blocks DNS
  rebinding), and requires a per-launch token that other sites can't read.
- **Honest progress bar for a non-deterministic agent.** It blends the agent's
  self-reported milestones, a floor that rises with each tool call, and a time curve
  from the median duration of past runs. It never moves backwards and stays under
  100% until the run really ends.
- **Faster, cheaper queries.** My first live case took ~30 warehouse queries, about half
  of them wasted. I wrote a staged query playbook (scoped by date and region,
  parallelised, split on timeout) that the agent follows and updates as it learns
  new facts.

## Architecture

```
 Browser UI ──(localhost + token)──► Studio server (Node, zero deps)
                                        │  per-turn tool allowlist
                                        ▼
                               headless agent CLI (stream-json, resume)
                                        │  skill rules + query playbook
                     ┌──────────────────┼──────────────────┐
                     ▼                  ▼                  ▼
              data warehouse      knowledge search     document tools
              (read-only hook)    (read-only)          (approval turn only)
```

## Turn types and permissions

| Turn | Allowed |
|------|---------|
| Light | Skill, file read, knowledge search. **No database.** |
| Heavy | + read-only database, read-only documents |
| Approve | + document create/edit. **Only after human review, only for qualifying conclusions** |

## Impact
- Investigation memo in ~[5-15] min vs ~[30+] minutes by hand.

## What I learned
Putting an AI agent into production work is mostly about boundaries: what it can touch,
when, and how a human can check what it did.
