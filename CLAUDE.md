# medflow-bioinfo

Agent-driven bioinformatics workflow framework. Agents interpret scientific
protocols, compile executable DAGs, audit them, and dispatch independently
versioned analysis nodes. The repository contains workflow instructions and
metadata rather than a compiled orchestration program.

## Project Structure

```text
medflow-bioinfo/
├── .agents/skills/
│   └── medflow-cleanup/SKILL.md # Codex cleanup mirror
├── .claude/skills/
│   ├── medflow-compile.md     # Intent-first protocol → workflow.json
│   ├── medflow-audit.md       # Initial and revised-plan audit
│   ├── medflow-run.md         # Attempt workspaces and active plans
│   └── medflow-cleanup.md     # Explicit user-invoked cleanup only
├── protocols/
│   └── 9-node-breast-cancer.md
├── workflows/                # Generated workflow JSON files
├── nodes/                    # Fresh registry clones; gitignored
├── runs/                     # Runtime environments and outputs; gitignored
├── registry.yaml             # Authoritative node URLs and versions
└── subagent-test-prompt.md    # Clean-sandbox integration-test prompt
```

## Workflow Skills

| Skill | Input | Responsibility |
|-------|-------|----------------|
| `medflow-compile` | Intent-first protocol Markdown | Generate immutable schema-2.0 workflows with semantic step intent, pinned node revisions, and no runtime output paths |
| `medflow-audit` | Initial or proposed full plan | Verify initial plans, changed parameters, replacement nodes, dependencies, and complete revised downstream subgraphs |
| `medflow-run` | Schema-2.0 workflow or workflow-run registry | Create UUIDv4 attempt workspaces, dispatch only the active full plan, and record post-terminal selection, rerun, replacement, halt, or escalation |
| `medflow-cleanup` | Explicit user request and terminal workflow-run root | Dry-run and confirm eligible attempt deletion while protecting active and selected workspaces |

`medflow-audit` is mandatory before unrestricted execution. A deferred audit
permits only the prerequisite fetch steps named by the audit; repeat the audit
before downstream analysis.

Only workflow schema `2.0` is executable. A new workflow run creates one
timestamp-plus-random run root, one `workflow-run.json`, immutable complete plan
snapshots, and one UUIDv4 workspace per node attempt. After a run starts, agents
must dispatch only from `active_plan_id`, never directly from the original
`workflow.json`. Every terminal attempt workspace is immutable.

The audit may edit a pinned node checkout or write deterministic adapter/helper
code when scientific meaning and public contracts are preserved. Every repair
must be exported beneath `workflows/remediations/<workflow>/<finding-id>/`,
labeled `MEDFLOW_AUDIT_GENERATED_REMEDIATION`, checksummed, executable from a
recorded command/environment, tested, and re-audited. Never overwrite raw
inputs. Run remediated workflows only through the generated
`workflows/<name>.audited.json`.

## Protocol Contract

Protocols describe research intent, datasets, comparisons, scientific
preferences, semantic analysis stages, and quality gates. They do not need to
name node repositories, subcommands, CLI flags, filenames, or complete output
inventories. `medflow-compile` resolves those details from `registry.yaml`,
and freshly cloned node contracts. `registry.yaml` supplies clone coordinates;
node selection and parameter filling come from each pinned node's `SKILL.md`,
verified against its entry point. There is no central node manifest.

## Node Acquisition and Provenance

- Obtain nodes only by cloning their `registry.yaml` URLs.
- Clone each required node fresh for compilation and execution; do not reuse a
  previous `nodes/` checkout.
- Do not create symlinks or acquire node packages from directories outside the
  active sandbox.
- Registry versions are not commit pins. Compilation records the URL, resolved
  default branch, and exact commit SHA in every workflow step. Audit and run
  must use that same commit rather than resolving the remote again.
- Fresh acquisition replaces the target node directory. Preserve any local
  node work before invoking compile or run; do not treat `nodes/` as durable
  storage.

## External Inputs

Reference resources not produced by workflow nodes, such as a GSEA GMT file,
must appear in the compiled workflow's `external_inputs`. Record their source,
database/release, expected format, run-local destination, retrieval time, and
checksum. Never substitute a different release silently.

## Language

Use English for artifacts, commit messages, and agent communication.

## Successful-Run Reproducibility Code Bundle

Every MedFlow/IRE node successful run must produce a `code/` directory inside its output directory. Treat `code/` as an always-produced output artifact and document it in `SKILL.md`/workflow contracts with semantic type `reproducibility_code_bundle`.

Required contents:

- `README.md` - what was run, how to rerun it, and which generated outputs it corresponds to.
- `run_command.sh` - a rerun command template using the same subcommand and effective parameters, with placeholders for machine-specific paths and no secrets.
- `parameters.json` - effective parameter values after defaults and config binding.
- `inputs_manifest.json` - input paths as provided, file sizes, and hashes when practical; never copy raw input data into `code/`.
- `environment.yaml` or copied `envs/*.yaml` - declared runtime environment used by the node.
- `session_info.txt` - package/session/runtime information when available.
- `scripts/` - a snapshot of the entrypoint and helper scripts actually used for the run.

Generate the bundle from actual effective runtime values, not hand-written examples. The bundle must not contain credentials, API tokens, hidden local config, raw input data, or unrelated repository files. Output validation and tests must fail when a successful run omits `code/` or required bundle files.

## Version

**Current:** 1.0.0
**Updated:** 2026-07-08

### Release Version Decision

Determine the next release version from the net public-contract difference
between the release candidate and the latest version tag already published to
the remote repository. Only incompatibility with that pushed release changes
the major version number. Unpublished commits, local or draft version tags, and
ordinary branch pushes do not establish a new release-version baseline. Do not
assign a version bump independently to each intermediate commit.

| Net change relative to last published version | Bump |
|---|---|
| Internal refactor, tests, documentation, or correction of an unreleased intermediate change | none |
| Backward-compatible bug fix that restores documented or intended behavior | patch |
| Backward-compatible addition to inputs, outputs, parameters, or behavior | minor |
| Incompatible change to a previously published contract | major |

Commit prefixes describe individual changes but do not mechanically determine
the release version. Select the highest bump required by the final net change.
### Version Synchronization

The node release version must remain consistent across all authoritative version
locations, including:

- `SKILL.md` `version:`
- `CLAUDE.md` `Current:`
- runtime version constants
- package metadata
- registry entries and release tags, when applicable

Determine the release version once, then update every synchronized location to
that version. Synchronizing version fields does not trigger an additional bump.

If authoritative version locations disagree, the release is incomplete. Resolve
the mismatch before committing, tagging, or pushing.

A protocol or extension may use an independent version only when it is
explicitly identified as separate from the node release version.

### Procedure

1. Identify the last published version tag.
2. Review the net runtime and public-contract difference from that tag.
3. Select the highest applicable bump from the table.
4. Keep the existing version when the net change requires no bump.
5. Update all synchronized version locations.
6. Update the `Updated:` date.
7. Verify that all authoritative version values agree.
8. Commit the release version as `chore: bump to X.Y.Z`.
9. Push the version commit with the release changes.
