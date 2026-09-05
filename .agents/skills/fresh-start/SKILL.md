---
name: fresh-start
description: >-
  Safely produce a clean Firstmate fleet restart across project homes and Herdr.
  Use when the captain invokes /fresh-start, asks for a clean restart, wants to
  update Pi, Herdr, or Firstmate, or asks to reopen configured project homes.
  Start with a report-only inventory, preserve unlanded work, clean only through
  guarded completion proofs or fresh captain confirmation, and support a
  when-safe recheck without forcing timing.
user-invocable: true
metadata:
  internal: true
---

# fresh-start

This skill is the captain-facing procedure for restarting a Firstmate fleet cleanly while preserving work.
It coordinates existing records and guarded commands; it does not introduce a broad shutdown or cleanup command.

## Invocation and modes

Use `/fresh-start` or `/fresh-start report` for the default report-only inventory.
The report names every observed worker, home, project, exact durable endpoint, visible Herdr area, current work state, open PR, and unresolved safety condition.
Nothing is stopped, closed, deleted, cleaned, restarted, or updated by the report-only mode.

Use `/fresh-start confirm` only after reviewing the report and stating the concrete actions and homes that may change.
The confirmation must identify the bounded action, such as parking worker `alpha`, cleaning completed worker `beta`, updating Pi, or updating Herdr.
A general request to "clean everything" does not authorize an unknown target, a broad process kill, a merge, or a workspace close.

Use `/fresh-start when-safe` when the fleet is not yet safe to restart.
This records the remaining blockers and the exact bounded recheck rather than waiting indefinitely or forcing the window.
Use `/fresh-start when-safe recheck` after the expected condition changes, or run `/fresh-start` in a later Firstmate session to obtain a fresh inventory.

The mode is report-only unless the captain explicitly selects an action mode.
An action mode still stops at any ambiguous identity, unreadable record, unlanded worktree, active update, open captain decision, or missing completion proof.

## Safety contract

The durable Firstmate record is the authority for task ownership, project, home, endpoint, worktree, current state, and completion.
Herdr labels, tab titles, workspace titles, visible text, focus, and proximity are observation hints only.
Never close, delete, stop, kill, move, or clean anything based only on a visual Herdr label.
Herdr workspace, tab, and pane ids must be read from the selected named session and matched to durable Firstmate metadata before any action.
If an id, home, project, or endpoint cannot be matched exactly, preserve it and report it as unknown.

Never begin with `herdr server stop`, `herdr server reload-config`, `herdr server live-handoff`, `pkill`, `killall`, a process-name sweep, or another broad shutdown operation.
Never use a raw Herdr close or delete operation as a substitute for Firstmate cleanup.
Use `bin/fm-control.sh` for an exact recorded worker lifecycle action and `bin/fm-teardown.sh` for exact completed-work cleanup.
Those commands own endpoint identity, completion proofs, locks, worktree handling, and durable-record retirement.

Uncommitted or otherwise unlanded work is never discarded, overwritten, stashed, reset, force-pushed, or silently moved.
`--force` on `bin/fm-teardown.sh` is permitted only when the captain explicitly authorizes discarding the identified work, and it is never implied by fresh-start.
An open PR is not a merge and is not merge authority.
An open PR may remain open while `bin/fm-teardown.sh` cleans a worktree only if that command independently proves the work is safely landed under its own rules.
Do not merge, close, or alter a PR as a side effect of fresh-start.
Respect the project's configured merge authority and use `bin/fm-pr-merge.sh` only for a separately authorized merge.

An update is a fleet-impacting action and requires a safe completed-work proof or fresh explicit captain confirmation for the update scope.
Do not update a live tool while an active or unknown worker, captain decision, unlanded worktree, or unresolved external dependency could be affected.
If safety cannot be established, use the when-safe path.

## Phase 1: inventory

Always collect the report before proposing or taking an action.
Use the structured fleet view as the starting point:

```sh
bin/fm-fleet-snapshot.sh --json
bin/fm-fleet-view.sh
```

The snapshot is observational and its header owns the bounded cross-home collection behavior.
For each local task in the snapshot, use `bin/fm-crew-state.sh <task-id>` when a current state decision matters.
Read the exact `state/<task-id>.meta` record only as needed to confirm its `kind`, `project`, `home`, `worktree`, backend, endpoint, PR, and report fields.
For a registered secondmate, read `data/secondmates.md` and the secondmate home summary rather than treating the parent status history as current child state.
For a remote secondmate, preserve the route when its host or home is unavailable and report that the remote state is unconfirmed.

Inventory project homes from the authoritative home and project records before looking at Herdr.
Record the absolute home path, the project clone or project root, the task worktree, and the owning Firstmate instance for every worker.
Treat a worker without a matching durable record as unmanaged work that must be preserved and escalated.
Treat a durable record without a live endpoint as recoverable work, not permission to delete its local copy.

Inventory the live fleet's named Herdr session with read-only commands and the current CLI help.
Use the explicit fleet session, normally `default`, and capture JSON for the session, workspaces, tabs, and panes.
The command shape may vary by installed Herdr release, so consult `herdr --help` and the relevant subcommand help before running it.
The required read-only surfaces are `session list`, `workspace list`, `tab list`, `pane list`, and exact-id detail or metadata reads where available.
Record each workspace id, tab id, pane id, current working directory, session name, parent workspace id, agent identity, and display labels.
Use pane metadata, process information, and agent-session information to corroborate the durable endpoint rather than trusting a title.

If the live default session cannot be read, do not guess from `HERDR_SESSION`, focus, or a label.
Report Herdr as unconfirmed and leave every area untouched until the named session and exact ids can be read.
The captain fleet uses the running `default` session; a lab or test session must never be confused with it.

## Phase 2: classify

Classify each inventory row as `active`, `safe-to-park`, `safe-to-clean`, `waiting`, or `unknown`.
Keep the evidence and the next safe action beside each row so a single ambiguous row cannot disappear in a summary.

`active` means the worker is working, has unlanded changes, owns an unresolved decision, or has a live dependency that fresh-start could interrupt.
`safe-to-park` means its exact endpoint is known and stopping the agent will preserve the endpoint, local copy, durable instructions, and uncommitted changes.
`safe-to-clean` means the existing `bin/fm-teardown.sh` proof is expected to pass for that exact task, or the task is a completed scout with its report and decision gate satisfied.
`waiting` means a captain decision, review, upstream release, rate limit, host, credential, or other external condition remains.
`unknown` means any identity, state, record, endpoint, Herdr relation, or completion fact is unreadable or contradictory.

An open PR is `active` or `waiting` until its work is independently proven safe to clean.
Green checks, a pushed branch, or an attractive PR title does not authorize a merge.
A merged PR does not authorize cleanup until the exact local task and PR head are matched by `bin/fm-teardown.sh`.
A visible Herdr area with no exact durable match remains `unknown` and is preserved.

A project or home is restart-ready only when every row affecting it is either safely parkable or independently complete, no row is unknown, no captain call is unresolved, and no required host or credential is unavailable.
One unsafe project does not authorize a broad action on the rest of the fleet unless the captain explicitly bounds the other action to exact safe targets.

## Phase 3: preserve, finish, or park

Let active work finish when it is making safe progress and the update window is not urgent.
Do not interrupt a worker merely to make the report look clean.
If the captain chooses to park a local worker, use the exact task id with:

```sh
FM_HOME=<firstmate-home> bin/fm-control.sh <task-id> exit
```

`exit` preserves the endpoint, worktree, and every uncommitted change, and it is not cleanup.
Use it only after the captain confirms the exact worker or after the report supplies a safe completed-work proof for the bounded park operation.
For a restart in place, use `FM_HOME=<firstmate-home> bin/fm-control.sh <task-id> relaunch --note '<durable progress note>'` only for the exact task and only with the required confirmation or proof.
Never use `fm-send.sh` to deliver a lifecycle command.

For a local secondmate, use the secondmate-specific restart path owned by `bin/fm-update.sh` and `bin/fm-secondmate-restart.sh` so its own durable home and child work survive.
For a remote secondmate, operate through its configured host path and preserve the route when the host cannot prove a safe action.
Do not read a secondmate's chat as a substitute for its home records or routed status.

If a worker is in a project worktree with uncommitted changes, preserve the worktree and record the path and next action in the durable task record.
Do not reset it, return it to a pool, or remove its pane.
If the work is ready for review, preserve the PR and report its full URL without merging it.

## Phase 4: clean safe completed work

Clean only exact task ids already classified as `safe-to-clean`.
Run the existing guarded command for each task separately:

```sh
FM_HOME=<firstmate-home> bin/fm-teardown.sh <task-id>
```

Let `fm-teardown.sh` perform the complete landed-work, scout-report, endpoint, worktree, backlog, and Herdr exact-pane checks.
If it refuses, preserve all records and local work, report the concrete refusal, and use `/fresh-start when-safe` if the condition is expected to clear.
Never work around a refusal with `--force`, raw Git cleanup, raw Herdr close, or a process kill.
Use `--force` only after the captain explicitly says to discard the exact identified work, and state what will be discarded before running it.

Cleaning one completed task does not authorize cleaning another task with a similar label, neighboring pane, same project name, or same home.
The Herdr presentation journal is never cleanup authority.
Firstmate cleanup may close only the exact endpoint recorded for the task, and it must never close a workspace as a visual convenience.

After each guarded cleanup, refresh the inventory and confirm that the exact endpoint and durable record changed as expected before proceeding.
Do not report a clean restart while any worker, project home, open PR, captain call, or visible area remains unconfirmed.

## Phase 5: update tools through supported paths

Update tools only after the inventory and preservation phases are complete for the bounded update scope.
Review the installed command's current help immediately before an update because vendor flags and handoff behavior can change.

Update Firstmate by invoking `/updatefirstmate`.
That skill owns the fast-forward-only update of the primary and configured secondmate homes, the restart or reread split, and the recovery of skipped homes.
Do not replace it with `git pull`, force, stash, or a manual secondmate restart.

Update Pi through its supported vendor command, normally `pi update`.
Do not run it while a live Pi worker could be interrupted unless the captain explicitly confirms that bounded update.
After a Pi update, reopen affected Firstmate homes through their durable records and run the normal session-start procedure once per home.

Update Herdr through its supported vendor command, normally `herdr update` or the exact current command shown by `herdr update --help`.
Herdr update is server-wide for the selected installation, so it requires a fresh captain confirmation unless every affected session and worker has a safe completed-work proof.
Do not invoke Herdr server lifecycle or update operations through a lab helper, and never use the lab session as a proxy for the captain's live `default` session.
Do not use `--handoff` unless the captain explicitly selects it after the current help and active-session impact are understood.

For any validation experiment, establish the lab contract before provisioning:

```sh
HERDR_LAB_HELPER=/Users/avitoshtotaram/github/kunchenguid/firstmate/bin/fm-herdr-lab.sh
HERDR_LAB_SESSION=$("$HERDR_LAB_HELPER" name fresh-start-skill-v1)
export HERDR_LAB_HELPER HERDR_LAB_SESSION
fresh_start_lab_teardown() {
  local status=$?
  "$HERDR_LAB_HELPER" teardown "$HERDR_LAB_SESSION" || status=$?
  trap - EXIT
  exit "$status"
}
trap fresh_start_lab_teardown EXIT
"$HERDR_LAB_HELPER" provision "$HERDR_LAB_SESSION"
```

Use only the generated non-`default` `fm-lab-*` session.
Route every task-specific non-lifecycle Herdr command through `"$HERDR_LAB_HELPER" run "$HERDR_LAB_SESSION" ...` so the helper appends the explicit trailing `--session`.
Use the helper for provisioning and teardown, never direct Herdr server or session lifecycle commands.
Do not use direct session deletion, direct session stopping, or ambient `HERDR_SESSION` isolation.
The helper's before/after default-session tripwire must remain intact, and the live captain fleet remains out of scope for lab cleanup.

If a vendor update cannot complete without stopping an unknown or active area, do not force it.
Record the version, affected area, reason it is unsafe now, and the exact update command to recheck later.

## Phase 6: reopen configured project homes

Reopen only homes identified in the inventory and only through their configured Firstmate entrypoint.
Keep each home's `FM_HOME` pointed at that home's durable data, state, config, and project roots.
Do not open a project clone as though it were a Firstmate home, and do not create a duplicate home because a path or label is inconvenient.

For each reopened home, run its standard session-start entrypoint exactly once so it can recover the durable queue, records, configured projects, and supervision instructions.
Use `FM_HOME=<home> <firstmate-code-root>/bin/fm-session-start.sh` for a home whose code root is known, and let that command emit the home's recovery and supervision instructions.
Let the home's own startup and update paths decide whether a secondmate or worker needs a safe relaunch.
If a home was skipped because it was dirty, diverged, offline, or otherwise unconfirmed, leave it in place and report the reason rather than reopening it through a substitute path.

Reopening a home does not reopen, resume, or attach to a private vendor conversation.
Durable briefs, task records, project records, and configured homes are the source of truth for pickup.
Use `bin/fm-control.sh ... relaunch` for an exact local worker that needs a new process, not an unverified vendor resume shortcut.

## Phase 7: resume and verify

After reopening, run a fresh report-only inventory.
Confirm that each preserved worker or home has the same project, home, worktree, endpoint relationship, and unlanded work as before the restart.
Confirm that each cleaned task has a durable completion transition and no remaining exact endpoint record.
Confirm that open PRs remain open unless a separately authorized merge occurred outside fresh-start.
Confirm that Herdr areas are either matched to exact durable records or explicitly reported as unknown and untouched.

Tell the captain what resumed, what remains parked, what remains waiting, what was cleaned by guarded proof, and what was not changed.
Include the next action for every unsafe or unknown row.
Do not call the operation complete while any required safety fact is unconfirmed.

## Timed and when-safe recheck

When `/fresh-start when-safe` finds active or unknown work, do not sleep, poll indefinitely, or force a transition.
Record a compact durable note containing the inventory timestamp, exact home and project, blocker, owner of the next action, and the command or invocation that rechecks it.
Keep an existing task note or captain-held backlog item as the record when one exists; do not create a parallel tracker for a one-off wait.
If no suitable durable record exists, report the note in the captain response and ask the captain to invoke `/fresh-start when-safe recheck` after the stated condition changes.

The recheck repeats the full inventory and reclassification phases before any action.
An upstream release, host recovery, rate-limit reset, or captain answer clearing a blocker is not proof that another row is safe.
If the blocker clears without a captain response, preserve the prior note, record how it cleared, and continue from the fresh report.
If it remains unsafe, update the same durable note with the new evidence and leave the fleet untouched.

## Captain-facing outcome

The concise completion report should say which exact homes were inventoried, which work was preserved or parked, which completed tasks `fm-teardown.sh` cleaned, which supported updates ran, which homes reopened, and which projects resumed.
It should separately name open PRs, unlanded work, unresolved decisions, unknown Herdr areas, skipped homes, and the next safe recheck.
It must never imply that an open PR was merged, that a label identified ownership, or that a partial update was a complete fleet restart.
