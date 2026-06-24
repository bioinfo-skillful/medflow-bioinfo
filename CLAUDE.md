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
