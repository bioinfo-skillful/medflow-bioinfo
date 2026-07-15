# medflow-bioinfo

Agent-driven bioinformatics workflow framework. MedFlow turns intent-first
scientific protocols into audited, executable multi-node workflows without a
hardcoded DAG or compiled orchestration program.

## How It Works

1. **Protocol** — describe the research question, datasets, comparisons,
   scientific preferences, semantic analysis stages, and quality gates.
2. **Compile** — `medflow-compile` clones current node contracts from
   `registry.yaml`, maps protocol intent to compatible capabilities, and writes
   `workflow.json` with immutable node commit SHAs, explicit file bindings, and
   external inputs.
3. **Audit** — `medflow-audit` checks configuration, data compatibility, sample
   identifiers, node revisions, and runtime dependencies. Deferred checks allow
   only prerequisite fetch steps before the audit is repeated. Safe repairs may
   edit a pinned node checkout or create run-local adapter code, but every
   repair must be labeled, bundled, reproducible, tested, and re-audited.
4. **Run** — `medflow-run` freshly clones and checks out the compiled node SHAs,
   executes the audited DAG in isolated per-step environments, and performs an
   automatic agent review after every node attempt. A failed review triggers a
   reproducible diagnose-adjust-rerun loop before downstream execution.

## Project Structure

```text
medflow-bioinfo/
├── .claude/skills/           # Compile, audit, and run instructions
├── protocols/
│   └── 9-node-breast-cancer.md
├── workflows/                # Generated workflow JSON files
├── nodes/                    # Fresh registry clones; gitignored
├── runs/                     # Runtime environments and outputs; gitignored
├── registry.yaml             # Node Git URLs and declared versions
├── CLAUDE.md                 # Agent-facing project instructions
└── subagent-test-prompt.md   # Clean-sandbox integration-test prompt
```

## Available Nodes

| Node | Subcommands | Purpose |
|------|-------------|---------|
| `geo-microarray-processing` | `fetch`, `qc`, `clean`, `validate-input`, `validate-output` | Fetch and prepare GEO microarray expression, metadata, and group assignments |
| `batch-correction` | `intersect` | Intersect gene-level datasets, apply ComBat, perform PCA/boxplot QC, and emit a reproducibility bundle |
| `differential-analysis` | `run`, `validate-input`, `validate-output` | Differential analysis with count- and normalized-expression methods, figures, and reproducibility code |
| `univariate-filter` | `run` | Screen features with binary, multi-group, continuous, survival, or correlation models |
| `ml-feature-selection` | `sequential`, `parallel` | Select features using Random Forest, LASSO, consensus, and convergence gates |
| `diagnostic-model` | `train`, `valid`, `evaluate`, `visualize` | Fit and internally assess multivariable logistic diagnostic models |
| `go-kegg-enrichment` | `enrich`, `merge`, `plot-bar`, `plot-bubble`, `plot-net`, `plot-emap` | Perform GO/KEGG over-representation analysis and visualization |
| `gsea-enrichment` | `run`, `filter` | Perform preranked GSEA from differential-expression ranks and a GMT database |
| `external-validation` | `validate`, `dca` | Apply a locked model to an independent expression cohort and assess transportability |

## Reference Protocol

The intent-first [nine-node breast-cancer protocol](protocols/9-node-breast-cancer.md)
compares ER-positive and ER-negative tumors using two GPL96 discovery cohorts
and a separate GPL570 validation cohort.

"Nine-node" refers to nine distinct registered node package types. The DAG has
11 step instances because the GEO preparation capability is instantiated once
for each of the three cohorts.

```mermaid
flowchart TD
    A[Prepare GSE25066] --> C[Harmonize discovery cohorts]
    B[Prepare GSE20194] --> C
    C --> D[Differential expression]
    D --> E[GO and KEGG enrichment]
    D --> F[Preranked GSEA]
    D --> G[Univariate screening]
    G --> H[Feature selection]
    H --> I[Diagnostic model]
    I --> J[External validation]
    K[Prepare independent GSE21653] --> J
```

- **Comparison:** ER-positive relative to ER-negative
- **Discovery:** GSE25066 and GSE20194, GPL96
- **External validation:** GSE21653, GPL570; excluded from all training choices
- **Branches:** GO/KEGG enrichment, preranked GSEA, and diagnostic-signature
  development with independent validation

Recompile the protocol before execution. Historical seven-node workflows and
reports describe earlier runs and are not execution specifications for the
nine-node protocol.

## Registry and Reproducibility

- `registry.yaml` is the authoritative source for node Git URLs and declared
  versions.
- Every freshly cloned node's pinned `SKILL.md` is the authoritative source for
  capabilities, parameters, defaults, inputs, outputs, discovery behavior, and
  conditional results. The agent builds the needed catalog at compile time and
  verifies it against the node entry point. The workflow records the contract
  hash plus each filled parameter's source and rationale.
- Declared registry versions are not commit pins. Compilation resolves the
  current remote default branch and persists its exact commit SHA in the
  workflow. Audit and run must verify or check out that same SHA even if the
  remote branch advances later.
- Nodes must be cloned fresh from registry URLs. Existing `nodes/` contents are
  not a reproducible source and may contain local work.
- External reference data must be declared in `external_inputs` with release
  provenance and checksums.

## Audit Remediation

`medflow-audit` may repair contract-preserving implementation defects and write
deterministic input adapters. It never overwrites raw inputs. Every repair is
stored under `workflows/remediations/<workflow>/<finding-id>/` with:

- `label: MEDFLOW_AUDIT_GENERATED_REMEDIATION`;
- base node URL/version/commit and original workflow checksum;
- runnable source code or a complete node patch;
- raw-input, code, patch, and output checksums;
- exact commands, environment, seeds, logs, tests, and re-audit evidence.

Repairs that change scientific meaning or the public node contract still
require explicit user confirmation. A remediated audit writes a separate
`workflows/<name>.audited.json`; the original compiled workflow remains
unchanged.

## Per-Node Run Review

Every node attempt receives a unique workspace ID and is written beneath
`runs/<workflow>/workspaces/<workspace-id>/` with a labeled
`agent-review.json`. The agent reviews exit status, logs, declared
and observed outputs, hashes, schemas, dimensions, quality gates, scientific
sanity, downstream compatibility, and generated figures. A step is complete
only when this review passes or passes with documented acceptable warnings; a
zero exit code is insufficient.

Unexpected but contract-valid results are accepted with evidence, rationale,
and interpretation limits rather than rerun automatically. When review truly
fails, the agent makes the smallest safe correction and reruns into
a new attempt directory. Code edits and adapters go through the labeled audit
remediation workflow and re-audit. The agent asks the user only when a proposed
change affects scientific meaning or the public node contract, or when the
failure cannot be resolved within the sandbox. All attempts and corrections
remain in the final report.

The agent persists all workspace events in `workspace-registry.jsonl` and holds
the explicit winning workspace for each logical step in an atomic
`workspace-selections/<step-id>.json` record, aggregated into
`selected-workspaces.json` for reporting. Downstream nodes never infer or
consume the latest attempt. Each workspace contains its own fresh pinned node
checkout. Node environments may be shared by compatible pinned revisions and
environment-specification hashes through a separate `env_id`; outputs and
review artifacts remain isolated per workspace.

Selections are atomic and revisioned. A later attempt can supersede an accepted
workspace only after its own passing review and a recorded supersession event;
a failed later replicate instead opens a reproducibility finding. Shared
environments become immutable after first use, so dependency repairs receive a
new environment ID rather than altering prior execution state.

Direct argument, path, environment, configuration, or binding corrections are
recorded in `run-adjustment.json` with exact old/new values and provenance.
Rendered figure-review evidence is preserved and hashed. Corrections that
generate code, transform data, rewrite binding logic, or edit a node must use
the full audit-remediation bundle instead.

## Integration Constraints

- GSEA requires a human GMT gene-set database that is not produced by another
  workflow node. Compilation must resolve or request the exact source and
  release before execution.
- External validation requires an independent cohort, the locked selected-gene
  list, and raw logistic beta coefficients. Odds ratios are not valid scoring
  coefficients. Under the current node contract, every locked model gene must
  exist in the external matrix; a missing gene stops validation rather than
  permitting refitting, imputation, or silent feature removal.
- At the currently inspected external-validation revision, the implementation
  writes a raw linear score under a probability column without applying an
  intercept or inverse-logit transformation. Treat probability calibration and
  decision-curve claims as unsafe until that node implementation is corrected.
- KEGG results and some GSEA pathway plots are conditional on external service
  or optional-package availability; their absence must be reported explicitly.

## Language

Use English for artifacts, commit messages, and agent communication.
