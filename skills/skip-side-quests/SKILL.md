---
name: skip-side-quests
description: Keep implementation focused on the agreed goal; defer side issues and stop at major changes or environmental blockers. Invoke manually.
disable-model-invocation: true
metadata:
  opencode/autoinvoke: "false"
---

Keep the current work aimed at the agreed outcome and plan.

- Continue work that is required to complete and validate the agreed outcome. A problem that would make the result incorrect or prevent required validation is not incidental.
- Do not investigate or fix an incidental problem that is unnecessary for that outcome. Keep it in a session-local Deferred Findings list and report only the observed symptom, known impact, and a next action supported by facts already available. Do not create an issue, TODO, file, or comment unless asked.
- Make local, reversible implementation adjustments when they preserve the agreed outcome.
- Stop before a material change to accepted behavior, acceptance criteria, core architecture, security boundaries, target systems, or overall scope. Report the problem and ask for direction.
- For permission, authentication, network, tool, or external-state blockers, perform only bounded read-only diagnosis, bounded ordinary retries, and already-authorized alternatives. Do not change permissions, credentials, persistent configuration, or route around the blocker. Stop and report it.
- If classification is materially ambiguous, ask the user. Otherwise, decide and continue.
