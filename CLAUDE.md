# medflow-bioinfo

Agent-driven bioinformatics workflow framework. No compiled code — agents execute workflows guided by slash-command skills.

## Project Structure

```
medflow-bioinfo/
├── .claude/commands/          # Slash command skills
│   ├── compile-workflow.md    # Protocol .md → workflow.json
│   └── run-workflow.md        # Execute workflow.json
├── protocols/                 # Reference protocol .md files
├── workflows/                 # Generated workflow.json files
├── nodes/                     # Cloned node packages (git repos)
└── runs/                      # Workflow execution outputs
```

## Available Commands

| Command | Purpose |
|---------|---------|
| `/compile-workflow <protocol.md>` | Compile protocol to workflow.json |
| `/run-workflow <workflow.json> [--live]` | Execute workflow |

## Language

English — all artifacts, commit messages, and agent communication.

## Registry

Available node packages are discovered by scanning `nodes/` for `SKILL.md` files. Each node is a standalone git repository with the IRE node package format (SKILL.md, envs/, scripts/).


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

Bump rules — check commit messages since last tag before every push:

| Commit prefix | Bump | Example |
|--------------|------|---------|
| `fix:` | patch (Z) | 1.0.0 → 1.0.1 |
| `feat:` | minor (Y) | 1.0.0 → 1.1.0 |
| `BREAKING:` or SKILL.md input/output change | major (X) | 1.0.0 → 2.0.0 |
| `chore:`, `docs:`, `refactor:`, `test:` | none | — |

Procedure:
1. Before push, scan commits since last version bump.
2. Pick the highest bump level among them.
3. Update `Current:` and `Updated:` above.
4. Commit as `chore: bump to X.Y.Z`, push together.

