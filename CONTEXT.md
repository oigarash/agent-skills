# Scope Discipline

This context defines the language shared by the `touch-grass` and `skip-side-quests` skills. It separates decisions that need user input from work an agent should handle or defer.

## Language

**Agreed outcome**:
The requested result together with its accepted plan and acceptance criteria.
_Avoid_: Original request, task in general

**User-owned decision**:
A choice that materially changes requirements, UX, design, acceptance criteria, external behavior, cost, risk, or hard-to-reverse architecture.
_Avoid_: Every design decision, implementation preference

**Implementation decision**:
A routine, reversible technical choice that preserves the agreed outcome.
_Avoid_: User preference, material plan change

**Side quest**:
An incidental finding whose investigation or repair is unnecessary to complete and validate the agreed outcome.
_Avoid_: Required work, environmental blocker

**Environmental blocker**:
A permission, authentication, network, tool, or external-state condition that prevents progress through the agreed and authorized path.
_Avoid_: Inconvenience, ordinary implementation problem

**Material plan change**:
A change to accepted behavior, acceptance criteria, core architecture, security boundaries, target systems, or overall scope.
_Avoid_: Local implementation adjustment
