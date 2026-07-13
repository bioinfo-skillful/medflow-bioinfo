You are in a clean sandbox with no prior context. Start from scratch.

Compile, audit, and run the MedFlow protocol at `<PROTOCOL_PATH>`.

Interpret the protocol's scientific intent as written. Resolve node packages,
subcommands, arguments, defaults, bindings, conditional outputs, and external
resources from `registry.yaml` and freshly cloned node `SKILL.md` contracts.
Use the registry only for clone coordinates. Let the agent fill node choices,
subcommands, parameters, defaults, bindings, and outputs from each pinned
contract after verifying its entry point. Do not assume that the protocol
author knows node implementation details, and do not use a central manifest.

Obtain every required node by cloning its `registry.yaml` URL fresh. Do not
reuse an existing `nodes/` directory or a historical compiled workflow. Record
the remote URL, resolved default branch, and commit SHA for every node used.
Require compilation to persist that SHA and require audit/run to verify and
execute exactly it even if the remote branch changes later.

Run `medflow-audit` after compilation. If it returns deferred checks, execute
only the prerequisite fetch steps identified by the audit, repeat the audit on
their outputs, and proceed only after it passes.

The audit may edit pinned node code or write deterministic adapter/helper code
to remediate a finding. Preserve raw inputs. Store every repair in a labeled
`MEDFLOW_AUDIT_GENERATED_REMEDIATION` bundle containing the base commit,
original workflow checksum, source or complete patch, exact command,
environment, seeds, input/code/output checksums, tests, and re-audit evidence.
If remediation succeeds, run only the generated audited workflow. Changes to
scientific meaning or public contracts still require explicit user approval.

For external reference resources, use the compiled `external_inputs` entries.
Record source, database/release, retrieval time, and checksum. Stop rather than
guess when access, licensing, or resource identity is unresolved.

Report:

- the interpreted workflow and assumptions;
- audit status and findings;
- status, runtime, and warnings for every step;
- every node attempt's `MEDFLOW_NODE_RUN_AGENT_REVIEW` decision, evidence,
  accepted-warning rationale, figure inspection, adjustments, retry count, and
  unique workspace ID, parent workspace ID, shared environment ID, and selected
  workspace;
- all files produced;
- declared-versus-produced output mismatches;
- external resources and provenance;
- remediation finding IDs, labels, bundle paths/checksums, code or patch files,
  tests, and reproduction commands;
- excluded samples and failed quality gates;
- whether each failure is computational, scientific, environmental, or due to
  external-source drift.

Sandbox rules:

- Never reference, search, or read from a directory outside this sandbox.
- Git clone from `registry.yaml` URLs is the only way to obtain node packages.
- Never copy or symlink node packages from another location.
- Never create symlinks.
- Do not silently substitute a different node revision, external database
  release, dataset, method, or group orientation.
