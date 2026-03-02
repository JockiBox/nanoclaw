# Dockerfile Modification Intent

## Changes

1. Added `jq` to apt-get install block — needed by the asana CLI script to parse JSON responses from the Asana API.
2. Added `COPY bin/ /usr/local/bin/` after `RUN npm run build` — installs the CLI scripts into PATH so the container agent can call them by name.
3. Modified entrypoint to source `/workspace/env` if it exists — this loads service tokens (ASANA_ACCESS_TOKEN, etc.) into the environment for CLI scripts, while explicitly excluding API secrets (ANTHROPIC_API_KEY, CLAUDE_CODE_OAUTH_TOKEN) which are handled separately via stdin.

## Invariants

- `jq` must be in the existing apt-get block (not a separate RUN layer) to keep the image small
- `COPY bin/` must come after WORKDIR /app but before workspace directory creation (ordering relative to other COPY steps)
- The bin/ scripts must be executable at authoring time — COPY preserves permissions
- Entrypoint sourcing must exclude ANTHROPIC_API_KEY and CLAUDE_CODE_OAUTH_TOKEN to maintain the existing security model where those secrets only go through stdin
- If `/workspace/env` doesn't exist, entrypoint continues normally (|| true)
