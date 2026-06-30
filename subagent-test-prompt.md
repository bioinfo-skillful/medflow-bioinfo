You are in a sandbox with no prior context. Start from scratch.

Compile, audit, and run the MedFlow protocol at `<PROTOCOL_PATH>`.

Use the protocol configuration as written. Obtain every required node by
cloning its `registry.yaml` URL fresh; do not reuse an existing `nodes/`
directory.

Report the status of each step, all files produced, and any output mismatches.

Sandbox rules:
- Never reference or read from any directory outside this sandbox.
- Git clone from `registry.yaml` URLs is the only way to obtain nodes.
- Never create symlinks.
