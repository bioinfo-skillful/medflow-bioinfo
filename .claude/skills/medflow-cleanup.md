---
name: medflow-cleanup
description: Use when a user explicitly asks to inspect or remove terminal MedFlow attempt workspaces
category: Workflow
tags: [workflow, cleanup, provenance]
---

# MedFlow Cleanup

Clean terminal MedFlow attempt workspaces only after an explicit user request.
Never invoke this skill from `medflow-run`, audit, compilation, recovery, or an
automatic post-workflow hook.

## Input

Require an explicit workflow-run root:

```text
runs/<workflow-name>/<workflow-run-id>/
```

Load `workflow-run.json` and its active full plan before inspecting candidates.
Reject schema versions other than `2.0`.

## Safety Gate

Refuse all cleanup when the workflow registry says `running` or the workflow
`.medflow-running.lock` exists. After the workflow is terminal, refuse every
attempt whose workflow-registry record says `running` or whose workflow-owned
`attempt-locks/<attempt-uuid>.lock` exists.

Report missing, inconsistent, or apparently stale locks. Never clear a lock,
finalize an interrupted attempt, repair a registry, or infer that execution is
dead from directory age alone. Direct the user to resume `medflow-run` for
recovery.

The one exception is recovery of cleanup's own interrupted transaction, never
an execution lock. The mutation lock records operation, transaction ID, owner
nonce, host/process identity, creation time, and heartbeat. On an explicit
user request naming that transaction ID, verify the recorded owner is no longer
active, show pending tombstones and physical state, and require confirmation to
resume. Atomically append a takeover owner to the same lock record; do not
silently remove or recreate it. Only `medflow-cleanup` may finish that
transaction and release the lock after durable completion.

## Dry Run

Always produce a dry-run inventory before deletion. For each attempt report:

- workflow, step, attempt UUID, path, terminal status, and disposition;
- selected, stale, or unselected state;
- active-plan and dependency references;
- size, recorded output hashes, and existing cleanup tombstone;
- eligibility and the exact reason for protection or exclusion.

Classify candidates as follows:

- active or running: never eligible;
- terminal and unselected: eligible after explicit confirmation;
- selected: protected by default;
- inconsistent or missing provenance: not eligible.

Selected workspaces require a separate explicit force request naming each
attempt UUID. Before accepting force, report every active-plan, downstream,
audit, and result reference that will lose its physical workspace.

## Confirmation and Deletion

Ask the user to confirm the exact attempt UUIDs and estimated bytes from the
dry run. A general approval to clean is not enough. Re-read registries and locks
after confirmation. Atomically acquire the exclusive workflow-level
`.medflow-mutation.lock` with operation `cleanup` before that final re-read,
and hold it until all tombstones and transaction state are durable.
`medflow-run`, run recovery, and cleanup must all acquire this same mutex
before any registry, plan, selection, lock, or workspace mutation. Abort if the
mutex cannot be acquired or any status, selection, plan, or dependency changed.

Canonicalize each target without following a symlinked workspace. Require a
valid UUIDv4 that matches the registry and an exact path under
`<run-root>/workspaces/<step-id>/<attempt-uuid>/`. Reject symlinks, path
traversal, identity mismatches, and any target outside that containment. Never
repair or delete an invalid target.

Keep verified parent and target directory descriptors open and perform
descriptor-relative, no-follow deletion. Revalidate device and inode identity
at the destructive operation. If the platform cannot provide descriptor-anchored
deletion, refuse cleanup; a path-string check followed by recursive deletion is
not acceptable.

Write the workflow-level cleanup transaction and a durable `pending` tombstone
for each confirmed target before deleting any bytes. Each tombstone contains
attempt UUID, step, prior terminal status, disposition, effective parameters,
reason, timestamps, recorded checksums, canonical deleted path, confirmation
evidence, and cleanup transaction ID. Delete only those recorded attempt
directories; never delete the compiled workflow, active plan snapshots,
`workflow-run.json`, selected attempts not separately forced, shared
environments, node repositories, or external inputs.

After each successful deletion, atomically mark its tombstone `complete`. If
deletion is partial or the process is interrupted, stop, retain the `pending`
tombstone and incomplete transaction, and do not claim success. Release the
workflow mutation lock only after final transaction state is durable.

## Output

Report dry-run candidates, protected attempts, confirmed deletions, reclaimed
bytes, tombstone IDs, and any refused or incomplete operations. Never describe
a physically deleted workspace as reproducible unless its retained metadata and
external artifacts are sufficient for the narrower claim being made.
