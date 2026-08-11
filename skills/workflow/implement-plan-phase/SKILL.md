---
name: implement-plan-phase
description: Implement a single phase of a plan file, verify it, update plan progress, and commit — then stop.
argument-hint: <plan.md> <phase-number>
disable-model-invocation: true
---

# Implement plan phase

Execute exactly one phase of a written plan. `$ARGUMENTS` gives the plan file path and the phase number. The phase boundary is the finish line: the run ends with this phase implemented, verified, recorded in the plan files, and committed.

## 1. Load the phase

Read the plan file and locate the given phase and its task list. While reading, also note:

- the plan's Status line or progress markers,
- any verification steps the plan specifies for this phase,
- related plan docs the plan references (an index or overview file that tracks this plan's status).

If the plan file or the phase number can't be found, stop and ask.

## 2. Branch

Run `git branch --show-current`. If the current branch is already this plan's working branch, continue. Otherwise ask the user whether to create a new branch for this plan — suggest a name following the repo's branch-naming conventions — or stay on the current branch. Wait for the answer before editing anything.

## 3. Implement

Do the tasks in this phase, and only those. Work in later phases stays untouched even when nearby code invites it — mention the opportunity in the final report instead of taking it.

Done when: every task listed under this phase is implemented.

## 4. Verify

Run the verification the plan specifies for this phase; if it specifies none, run the repo's standard checks (tests, typecheck, lint). Fix failures introduced by this phase's work.

Done when: verification passes with this phase's changes in place.

## 5. Record progress

Update the plan file: check off this phase's completed tasks and refresh the Status line. Apply the same update to every related plan doc noted in step 1.

Done when: someone reading only the plan files can tell this phase is complete and which phase comes next.

## 6. Commit

Invoke the `conventional-commit` skill to commit the implementation together with the plan-file updates.

## 7. Stop

Report what was implemented, the verification results, and which phase comes next. The next phase begins only on a fresh instruction from the user.
