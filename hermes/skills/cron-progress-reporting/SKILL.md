---
name: cron-progress-reporting
description: Use when asked for periodic status updates during long work.
metadata:
  hermes:
    editorial_name: Progress Reporter
    editorial_description: Periodic status reports during long-running work via scheduled checks.
    requires_tools: [tool_call]
---

# Recurring Progress Reporting via Cron

An agent session cannot wake itself on a timer. When the user asks for periodic
updates ('report every N minutes') while long work runs, set one up as a
`cronjob_manage` reporter job instead of promising to check back yourself.

## Procedure

1. Say once, briefly: milestone updates as modules complete, plus a cron reporter
   for the interval — then create the job.
2. Create with `action=create`: a `schedule` like `every 5m`, `deliver=origin`
   (posts back to the originating chat), `workdir` set to the repo so context
   files load, and `continuity=true` so each run diffs against its own previous
   output.
3. Write the prompt self-contained: the job runs in a fresh session with no chat
   context, so include the repo path, branch, the exact commands to run
   (`git log --oneline -5`, `git status --short`, artifact counts), what counts
   as progress, the report language, and a max length. End with the rule: if
   nothing changed, emit one line saying so.

## Pitfalls

- **Cron prompts must be self-contained.** The run inherits no conversation —
  every path, command, and decision criterion goes in the prompt.
- **Keep the prompt ASCII-safe.** The scheduler rejects prompts containing
  invisible unicode (e.g. ZWNJ inside Persian text) — write the prompt in
  English and let the job render its *report* in the user's language.
- **Gate on change.** With `continuity=true`, instruct the job to compare against
  its previous output and stay to one line when nothing moved; otherwise every
  tick is noise.
