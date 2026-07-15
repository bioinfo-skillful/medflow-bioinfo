# medflow-bioinfo

Agent-driven bioinformatics workflow framework. Agents interpret scientific
protocols, compile executable DAGs, audit them, and dispatch independently
versioned analysis nodes. The repository contains workflow instructions and
metadata rather than a compiled orchestration program.

## Project Structure

```text
medflow-bioinfo/
├── .claude/skills/
│   ├── medflow-compile.md     # Intent-first protocol → workflow.json
│   ├── medflow-audit.md       # Pre-execution contract and data audit
│   └── medflow-run.md         # Audited execution with per-attempt agent review
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
| `medflow-compile` | Intent-first protocol Markdown | Clone node contracts from the registry, infer compatible capabilities and settings, and generate an execution-ready workflow |
| `medflow-audit` | Compiled workflow and available runtime data | Verify configuration, data compatibility, identifiers, node revisions, and environments; safely repair code/input adapters through labeled reproducibility bundles; return `pass`, `pass_with_remediation`, `fail`, or `deferred` |
| `medflow-run` | Audited workflow JSON | Give every node attempt a unique workspace ID, explicitly select accepted workspaces, share only compatible environments, agent-review each attempt, and safely rerun failures before downstream execution |

`medflow-audit` is mandatory before unrestricted execution. A deferred audit
permits only the prerequisite fetch steps named by the audit; repeat the audit
before downstream analysis.

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

## Release Version Policy

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
