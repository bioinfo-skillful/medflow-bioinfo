---
name: "Run Workflow"
description: Execute a workflow.json step by step, dispatching nodes via conda
category: Workflow
tags: [workflow, run, execute]
---

Execute a workflow from a workflow.json file.

**Input**: `workflows/<name>.json`. Default is real conda execution. Use `--stub` for dry-run (validate only, no dispatch).

## Steps

### 1. Load Workflow

Read `workflow.json`. Validate structure: steps, edges, and each step's `id`,
`node`, immutable `source` record, and `config`. The source record must contain
the registry URL, declared version, resolved default branch, and exact commit
SHA, plus the compiled `SKILL.md` contract hash. A legacy workflow without an
exact commit or contract hash must be recompiled.

**Prerequisite:** medflow-audit must run before execution.
- `pass`: proceed with the DAG.
- `pass_with_remediation`: run only the audit-produced
  `workflows/<name>.audited.json` after verifying all labeled remediation
  bundles.
- `fail`: do not execute any step.
- `deferred`: execute only prerequisite fetch steps identified by the audit,
  repeat medflow-audit on their outputs, and proceed only after it passes.

**If `remediations` is present:** Treat each entry as executable provenance,
not an informal suggestion. Before any affected step:

- require `label: MEDFLOW_AUDIT_GENERATED_REMEDIATION` in its
  `remediation.yaml`;
- verify the original workflow hash, pinned node base commit, raw-input hashes,
  patch/code hashes, environment record, and bundle checksum;
- reject missing labels, checksum drift, unrecorded working-tree changes, or a
  remediation that reports a scientific/contract change without recorded user
  approval;
- run only the recorded command from the recorded working directory and
  isolated environment;
- preserve raw inputs and write derived files only to the declared run-local
  remediation output directory;
- record the remediation finding ID and bundle checksum in step logs and final
  workflow provenance.

**If `file_bindings` is present:** Use it as the authoritative wiring map. `file_bindings` was produced by medflow-compile and contains exact file resolution instructions. The checklist items marked `[fb]` below can skip inference — use the binding directly.

**If `data_type_notes` is present:** Treat it as the compiler's expected data
type, then confirm it against a small runtime sample during the mandatory
pre-flight check. A mismatch requires re-audit; do not silently change methods.

**If `external_inputs` is present:** Treat it as the authoritative acquisition
and provenance map for reference resources that are not produced by workflow
nodes. Before dispatching the consuming step:

- acquire the declared resource into its run-local destination, or verify the
  declared local file if it was supplied by the user;
- verify its expected format and checksum when provided;
- record the source URL, database/release, retrieval time, and observed
  checksum;
- stop for user action when access requires unrecorded credentials, license
  acceptance, or an unresolved resource choice;
- never search outside the sandbox or substitute a different release silently.

### 2. Resolve Node Packages

For each step:
1. Parse `node` field: `<name>@<version>` and read the step's immutable `source`
   record. Do not resolve an unversioned node to "latest" at run time.
2. Read `registry.yaml` and confirm its URL matches `source.url`.
3. Read the freshly cloned node's `SKILL.md` for subcommands, parameters,
   defaults, inputs/outputs, file layout, discovery rules, and exceptions. Do
   not use or expect a central node manifest. Verify its SHA-256 equals the
   compiled contract hash before using any filled parameter.
4. **Always clone fresh from `source.url` inside the node-run workspace:**
   Generate the workspace ID as specified under Execute, create the workspace,
   and clone into its `node/` directory. Never share a mutable node checkout
   between attempts. Then detach at the compiled commit:

   ```text
   git clone <source.url> runs/<workflow>/workspaces/<workspace-id>/node
   git -C runs/<workflow>/workspaces/<workspace-id>/node checkout --detach <source.commit>
   git -C runs/<workflow>/workspaces/<workspace-id>/node rev-parse HEAD
   ```

   The observed HEAD must exactly equal `source.commit`. Clone for every node
   dispatch, including retries; never reuse another workspace's checkout or
   replace the pin with the current remote HEAD.

   For a declared `NODE_PATCH`, verify `node.patch` and its SHA-256, run
   `git apply --check`, apply the patch to this clean detached checkout, and
   confirm the resulting diff and file hashes exactly match
   `remediation.yaml`. Label the execution source as
   `pinned_commit_plus_audit_patch`; never execute an unrecorded dirty tree.
5. **CRITICAL: Sandbox integrity rules**
   - **Never reference, search, read, copy, rsync, or symlink from any directory outside this sandbox.** This includes `/work/run/projects/bio-13/test/IRE_product`, `../nodes`, `~/projects`, or any other path.
   - **Never use `cp -r`, `rsync`, or `ln -s` to obtain node packages** — always `git clone` from `registry.yaml` URLs.
   - **Never symlink files** — symlinks break sidecar file lookups (dirname resolution) and env file discovery.
   - If a node's git URL is unreachable, report the error and stop. Do not search for alternatives outside the sandbox.
6. **Read `SKILL.md`** to get the full node contract (parameters, entry point,
   environment file, inputs, outputs, layouts, and exceptions).
7. **Read the node's entry point** (`scripts/main.R` or `scripts/main.py`):
   - Treat `SKILL.md` as the declared contract and the entry point as observed
     implementation behavior.
   - Verify the implementation matches declared subcommands, parameters,
     `file_discovery`, `file_layout`, and conditional outputs. If they disagree,
     stop and report contract drift; do not silently choose one.
   - **Do not guess file layout requirements from directory structure alone**

### 3. Build Execution Plan

From the DAG edges, determine execution order (topological sort):
- Nodes with no incoming edges run first
- Nodes run when all upstreams complete
- Parallel branches run concurrently where possible

### 4. Execute (Stub)

If `--stub` is set:
- Validate all nodes resolved
- Validate all required config params present
- Validate edges form a valid DAG (no cycles)
- Print execution plan: what runs in what order with what args
- Report: "Stub run complete. N steps validated. Ready for live execution."

### 5. Execute

Default mode — real conda execution:

#### Node-Run Workspace Identity

Before every node dispatch, including the first attempt and every retry,
generate a globally unique `workspace_id`. Use a collision-resistant value
that includes the logical step ID, UTC timestamp, and random UUID component,
for example `deg-20260714T153012Z-3d49c1a8`. Never reuse a workspace ID, even
when a prior dispatch did not start successfully.

Create the node-run workspace at:

```text
runs/<workflow>/workspaces/<workspace_id>/
```

Place the fresh node checkout, command record, input/config hashes, logs,
outputs, review evidence, `agent-review.json`, and any `run-adjustment.json`
inside that workspace. Append
an entry before dispatch to
`runs/<workflow>/workspace-registry.jsonl` containing the workspace ID,
workflow/step/node identity, attempt ordinal, parent workspace ID for a retry,
status, paths, and creation UTC. Update status by appending a new event; never
rewrite registry history. Serialize registry appends when independent DAG
branches run concurrently so an event cannot be interleaved or lost.

The executing agent must hold the current candidate `workspace_id` throughout
dispatch and review. When a step has several workspaces, never infer the winner
from timestamps, directory order, or the largest attempt number. After review,
persist the explicit choice atomically in
`runs/<workflow>/workspace-selections/<step-id>.json`, containing the selected
workspace ID, review decision/hash, output-inventory hash, config/input hashes,
selection revision, and any superseded workspace ID. This per-step file is the
authoritative held choice. Serialize selection updates with downstream
dispatch so a consumer cannot race a selection change. Downstream bindings may
read outputs only from that selected workspace and must record its selection
revision. A retry creates a new workspace and leaves the prior selection
unchanged until the new workspace passes review. At final reporting,
aggregate the per-step selections into
`runs/<workflow>/selected-workspaces.json` without changing them.

Do not start another workspace for a step after it has passed unless the user
requested a replicate, the accepted review was invalidated, a downstream
finding points to that step, or a declared reproducibility check requires it.
A later workspace never silently replaces a selected one. Select it only after
its own passing review and append a `selection_superseded` event containing old
and new IDs/revisions to the workspace registry. A later failed replicate does
not become selected, but it raises a reproducibility finding; investigate
whether the earlier selection remains valid. If invalidated, append a
`selection_invalidated` event, stop/invalidate affected downstream work, and
rerun before continuing.

Environments are not workspaces and may be shared to avoid repeated package
installation. Derive an `env_id` from the node URL, pinned commit, environment
specification hash, platform, and accepted environment remediations; store it
at `runs/<workflow>/envs/<env_id>/`. Record the shared `env_id` in every
workspace. Never share an environment when any of those inputs differ, and
never place mutable node outputs in the shared environment. Treat a shared
environment as immutable after its first dispatch. Any package installation,
repair, or dependency change creates a new environment specification hash and
therefore a new `env_id`; never mutate an environment already referenced by a
workspace.

For each step in topological order, generate and complete a pre-flight
checklist before dispatching the node. After every node attempt, complete the
mandatory agent review below. A successful process exit alone does not finish
a step. Do not dispatch a downstream step until the review passes.

#### Pre-flight Checklist Template

For each step, create this exact checklist and work through it:

```
□ 1. SKILL.md read: inputs=[...], outputs=[...], parameters=[...]
□ 2. Entry point read: subcommands=[...], file_discovery: {recursive, pattern, sidecar?}
□ 3. Upstream files inspected with a recursive file listing
□ 4. File layout confirmed: nesting? sidecar files present?
□ 5. Data type checked: opened sample file, values are (integer counts | log2 | normalized)
□ 6. Method matched: using (limma | DESeq2 | edgeR) because data is (log2 | counts)
□ 7. Config complete: required params=[...], provided=[...], overrides applied=[...]
□ 8. Env ready: compatible shared env verified at runs/<workflow>/envs/<env-id>/
```

**Each `□` must become `✓` before dispatch. Do not proceed past an unchecked box.**

#### Checklist Item Details

**1. SKILL.md read:**
- Parse the YAML frontmatter
- List all `inputs`, `outputs`, `parameters` with their types and defaults
- Note `file_layout` and `file_discovery` if declared

**2. Entry point read:**
- Open `scripts/main.R` or `scripts/main.py`
- Find the subcommand dispatch section — list all valid subcommands
- Check `list.files` / `glob` / `Sys.glob` calls: recursive or flat?
- Check for sidecar file logic: does it look for companion files?
- Check for conditional output logic: `if (!is.null(meta))` — when does an output NOT get produced?

**3. Upstream files inspected:**
- List files recursively rather than using a flat directory listing:

  ```powershell
  Get-ChildItem -LiteralPath <upstream-outdir> -Recurse -File |
    Select-Object -ExpandProperty FullName
  ```

  ```bash
  find <upstream-outdir> -type f
  ```

- List every file with its full path
- If multiple upstreams, inspect each one separately

**4. File layout confirmed:**
- Compare SKILL.md `file_layout` against actual directory structure
- If `file_discovery.sidecar` declared, verify companion files exist
- If sidecar files are missing and they're `needed_for` an output, that output WILL be silently skipped
- **Never symlink files between directories** — this breaks sidecar lookups

**5. Data type checked:**
- Open a sample input file: `head -3` or `read.csv(..., nrows=5)`
- Check: are values integers (counts)? decimals around 0-20 (log2)? 0-1 (normalized)?
- Record the observed data type

**6. Method matched:**
- Log2 microarray data → `--method limma`
- Raw integer counts → `--method DESeq2` or `--method edgeR`
- Unknown → ask user, do not guess
- If the node's default method is wrong for this data type, override it in config

**7. Config complete:**
- List all required parameters (bind: config)
- List all provided values from workflow.json `config`
- Flag missing required params
- Flag params where the default is wrong for this data/question
- Apply overrides

**8. Env ready:**
- Check if a functional env already exists at `runs/<workflow>/envs/<env-id>/`:
  - Windows: check `Scripts/Rscript.exe` for R or `python.exe` for Python.
  - POSIX: check `bin/Rscript` for R or `bin/python` for Python.
- If exists and the binary is executable: skip creation, use it directly.
- If missing or broken: create fresh.
  ```bash
  conda env create -f <env-file> -p runs/<workflow>/envs/<env-id>
  ```
- After env creation, verify R packages are loadable. If a package is unavailable
  via conda (common with Bioconductor packages on Windows), install via Rscript:
  ```bash
  conda run -p runs/<workflow>/envs/<env-id> Rscript -e \
    'if (!require("<pkg>")) { BiocManager::install("<pkg>", update=FALSE, ask=FALSE) }'
  ```
  Check each package declared in `<env-file>` before declaring env ready.
- Use `--override-channels` or ensure `nodefaults` in env file channels
- Never use pre-existing global conda environments

#### Dispatch and Verify

After checklist is complete:

**Dispatch:**
```bash
conda run -p runs/<workflow>/envs/<env-id> --cwd <node-dir> Rscript scripts/main.R <subcommand> --arg1 val1 --outdir runs/<workflow>/workspaces/<workspace-id>/outputs
```

**Handle result:**
- Parse NDJSON stdout → extract result line, files, metadata
- Exit code: 0 = success, non-zero = check `exceptions` in SKILL.md

**Post-flight verification (MANDATORY):**
```
□ Outputs declared vs produced: SKILL.md says [X, Y, Z], disk has [actual files]
□ Any mismatch? If yes, check: conditional output? sidecar missing? wrong subcommand?
```
- Cross-reference the `SKILL.md` outputs with a recursive listing of `<outdir>`
- If a declared output is missing: read the node code to find the condition that produces it
- **Do not claim "node doesn't produce X" without first checking whether your setup broke a sidecar dependency**
- Report any mismatch with the specific condition found

#### Agent Review and Retry Gate

After **every** node attempt, the executing agent must review the attempt; do
not ask the user to perform this routine review. Write
`runs/<workflow>/workspaces/<workspace-id>/agent-review.json` with:

- `label: MEDFLOW_NODE_RUN_AGENT_REVIEW`, workspace ID, workflow and step IDs,
  attempt number and parent workspace ID,
  node URL/version/commit, command, config hash, input hashes, start/end UTC,
  exit code, and environment identity;
- declared versus produced files, recursive file inventory and hashes, schema
  and dimension checks, sample/feature counts, non-finite or empty results,
  warnings/errors, and conditional-output explanations;
- scientific and statistical sanity checks required by the protocol and node
  contract, including contrast direction, class labels, thresholds, leakage
  guards, convergence/quality gates, and obvious degenerate results;
- figure inspection findings for every generated figure. Open each figure (and
  render PDFs when necessary); check readability, labels, legends, clipping,
  empty panels, implausible scales, and consistency with tabular results;
- downstream compatibility and the review decision: `pass`,
  `pass_with_warnings`, `retry`, or `blocked`, with evidence and proposed
  action.

Preserve rendered PDF pages or review screenshots under
`workspaces/<workspace-id>/review-evidence/` and include their hashes in
`agent-review.json`. Exclude `agent-review.json` itself from its file-hash
inventory to avoid a self-referential checksum.

On `pass` or `pass_with_warnings`, atomically select that workspace ID in
`runs/<workflow>/workspace-selections/<step-id>.json`; downstream bindings must
read its exact `outputs/` directory. Do not use a symlink, an implicit "latest"
workspace, or overwrite an earlier workspace.
Use `pass_with_warnings` when an output is unexpected but remains valid under
the protocol, node contract, quality gates, and downstream input contract.
Record each accepted warning, the observed evidence, expected behavior, reason
it is acceptable, and any limitation on interpretation. Conditional outputs
that are legitimately absent and non-fatal package/service warnings belong in
this category when their documented conditions are satisfied. Do not rerun
merely to eliminate a cosmetic or scientifically acceptable warning.

On `retry`, diagnose the finding, make the smallest in-scope correction, and
rerun the node in a new unique workspace whose registry entry names the failed
workspace as its parent. Re-run the full agent review after the retry.
Permitted automatic corrections include environment
repair, contract-preserving configuration or file-binding correction, and the
labeled reproducible audit remediations defined by `medflow-audit`. Never edit
raw inputs or silently change biological contrast, statistical method,
normalization, identifier meaning, imputation, feature-selection meaning,
model target, output schema, or the public node contract.

For a direct environment, argument, path, config, or file-binding correction,
write `workspaces/<new-workspace-id>/run-adjustment.json` before dispatch.
Record the prior
review path/hash, correction type and rationale, exact old/new values, workflow
and config hashes, command, environment change, affected inputs, and the agent
identity/time. A pure correction selects or passes already declared artifacts;
if it generates code, transforms data, rewrites binding logic, or edits a node,
it is not direct and must use audit remediation. Include accepted direct
adjustments in final workflow provenance so the accepted invocation can be
reproduced without inference.

If a retry requires adapter/helper code or a node edit, invoke
`medflow-audit`, create a `MEDFLOW_AUDIT_GENERATED_REMEDIATION` bundle, re-audit
the affected step and invalidated downstream assumptions, and run only the
updated audited workflow provenance. Record the finding ID and bundle checksum
in every subsequent attempt review.

Continue the diagnose-adjust-rerun loop while a new safe corrective action is
available. Mark `blocked` and stop before downstream execution only when:

- the correction would change scientific meaning or the public contract and
  needs user confirmation;
- required data, credentials, licensed resources, or network access remain
  unavailable after applicable recovery attempts;
- the node cannot be made reproducible in the sandbox; or
- the same failure persists for three consecutive attempts with no new
  evidence or safe corrective action.

Never treat a warning as an automatic failure, or a zero exit code or existing
output files as an automatic pass. Report all attempts and explain why the
accepted attempt passed, passed with warnings, or became blocked.

**Inline transformation (when needed):**

If `file_bindings` for this step contains `extract_group_col` or `transform` directives:
1. Read the source file identified in the binding
2. Execute the transformation inline — write a minimal script, not a new node:
   - When the binding declares `allowed_values`, retain only rows whose group
     value is in that exact allowlist. Report counts for retained and excluded
     values. Apply this rule to direct extraction, coalescing, and
     `combine_rows` transformations.
   - **Single column:** `extract_group_col: "er_status"` → extract `sample_id` + group column:
     ```python
     import csv
     with open("merged_metadata.csv") as inf, open("sample_group_map.csv", "w") as outf:
         r = csv.DictReader(inf); w = csv.writer(outf)
         w.writerow(["sample_id", "group"])
         for row in r: w.writerow([row["geo_accession"], row["er_status"]])
     ```
   - **Coalesce with fallback chain (deterministic — compiled by medflow-compile):**
     ```json
     "extract_group_col": {
       "unified_name": "er_status",
       "sources": {"er_status": ["P","N"], "er_status_ihc:ch1": ["P","N"]},
       "fallback_chain": ["er_status", "er_status_ihc:ch1"]
     }
     ```
     → Try each column in `fallback_chain` order. For each row, use the first column that has a valid value (in `sources[column]` allowlist). Filter to rows with valid groups. Report unified counts.
   - `transform: "transpose"` → transpose matrix
   - `transform: "merge_columns"` → combine multiple columns
   - `transform: "filter_features"` → subset an expression matrix by an
     upstream candidate table using the binding's declared feature-ID columns.
     Report candidate count, matched count, missing identifiers, duplicates,
     and final matrix dimensions; never substitute fuzzy identifier matches.
3. Write the output file to the current step's outdir
4. Bind the transformed file to the downstream parameter
5. Report: "Inline transform: coalesced er_status + er_status_ihc:ch1 → sample_group_map.csv (P=461, N=319)"

When the transformation is supplied by an audit `INPUT_ADAPTER` or
`HELPER_CODE` bundle, use the recorded script and command rather than rewriting
it. Verify source-input hashes before execution and derived-output hashes after
execution, and include the remediation label/finding ID in the step report.

**Wire data to downstream:**
- Check `file_bindings` in workflow.json first — if present, use the declared source_step, prefer, and transform directives
- If no file_bindings: read upstream SKILL.md outputs vs downstream inputs, match by semantic_type
- Pass resolved file paths (not directories) for `param_type: file`

### 6. Report

For each step: status, workspace ID and agent-review decision for every
attempt, adjustments made, selected workspace ID, shared environment ID, files
produced, runtime, warnings, figure
findings, and any output mismatches. Final: workflow status, output directory,
key result files, total retries, and blocked findings.

## File Resolution Protocol

1. List all files recursively with `Get-ChildItem -Recurse -File` on
   PowerShell or `find <dir> -type f` on POSIX; never use a flat listing.
2. Match by: exact filename → semantic_type → format → file content inspection
3. Respect `file_layout` declarations in SKILL.md (nesting, sidecar)
4. **Never symlink, move, or copy files** — breaks sidecar assumptions
5. If ambiguous: report options and ask user

## Anti-Patterns

- **Do NOT** symlink files into staging directories — breaks relative path assumptions in node code
- **Do NOT** guess file layout from directory structure — read the node's actual code
- **Do NOT** use global conda environments — create fresh per-run envs under `runs/<workflow>/envs/`
- **Do NOT** use default method without checking data type compatibility
- **Do NOT** skip reading the node's entry point before dispatching
- **Do NOT** assume a node doesn't produce a file just because you didn't look for it — verify outputs against SKILL.md
