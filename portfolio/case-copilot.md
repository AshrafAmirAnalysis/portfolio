# Case Copilot: a browser assistant for CRM case handling

**Chrome extension (Manifest V3) · JavaScript · Dynamics 365 Client & Web API · jsdom test harness**

## The problem
Investigators on a trust & safety team handled each case through dozens of manual
steps in a CRM: reading the case, writing an L1 summary, picking a reply template,
composing a customer email, filling category fields and walking the case through a
multi-stage closing process. It was slow and repetitive, and the manual steps caused errors.

## My role
Self-initiated and built solo while working as an investigator. I designed the
product, wrote the specs and shipped 40+ versions based on feedback from daily use.
Built with AI-assisted development (Claude Code): I set the design, requirements and
safety rules, then reviewed, tested and verified every release in live use.

## What it does
- Reads the open case and turns it into ready-to-paste summaries, replies and links
- **One-button quick actions** the user sets up: note with decision → reply →
  customer email → send → field macro → closure. Only the decisions and confirmations
  are left to the human
- Records and replays field "macros" across the case's business process stages
- Builds a structured hand-off package for an AI-assisted investigation tool
  (see [Investigation Studio](investigation-studio.md))
- Team templates live in a shared spreadsheet, so non-developers can edit them and
  the tool can be set up for another team without code changes
- Side panel plus a pop-out "app window" mode with three experience levels
  (Guided / Expert / Classic)

## Engineering highlights
- **From UI automation to platform APIs.** The first version replayed clicks and
  keystrokes. On the live CRM it kept failing because of timing and inconsistent markup.
  I rebuilt it on the CRM's client-side form API and REST Web API. The DOM is used only
  for reading. This removed a whole class of failures.
- **Safety guards before any write.** Every write first checks that the record in
  front of the user matches the case the tool thinks it is on. After one incident,
  where a stale timestamp was written back to a case, I added a permanent denylist
  (SLA, timer and datetime fields are never recorded or replayed) and a post-replay
  watcher that restores fields cleared by form business rules.
- **Root-causing a flaky integration.** Auto-composing emails worked about 1 run in 10.
  I wrote diagnostic probes, read the vendor form's own event handlers, and found
  three separate causes: a server-side template step that took longer than my timeout,
  a rich-text editor that ignored programmatic values, and a focus issue where the
  editor stalled when the page held keyboard focus. I fixed each one, with timing traces
  built into the user-facing messages.
- **Tested without a sandbox.** No test environment existed, so I built a mock of the
  CRM's client API and a jsdom test harness: 350+ automated checks across flows,
  pickers, email composition and end-to-end macro replay.
- **Privacy by design.** History stays in the local browser profile with a retention
  limit. Text is masked and reviewed before it reaches any LLM.

## Architecture

```
 ┌──────────────── Browser ────────────────┐
 │  CRM web app (case form, email form)    │
 │        ▲ read (DOM)   ▲ write (API)     │
 │        │              │                 │
 │  ┌─────┴──────────────┴──────────────┐  │
 │  │ Case Copilot extension            │  │
 │  │  side panel / pop-out app window  │  │
 │  │  flows · macros · email · guards  │  │
 │  └─────┬───────────────┬─────────────┘  │
 └────────┼───────────────┼────────────────┘
          │               │
   shared config     hand-off package
   spreadsheet       → Investigation Studio
```

## Impact
- Per-case handling: ~[10] min → ~[2] min 
- Closing macro runs end to end in ~16 seconds, versus [~20] manual steps
- Used daily in live operations

## What I learned
Integrating with a platform you don't control means probing it, reading its code
and building guardrails, not trusting it. Simplicity also won: users asked for
*fewer* buttons, not a dashboard.
