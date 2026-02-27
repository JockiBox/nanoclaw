# Dockerfile Modification Intent

## Changes

1. Added `jq` to apt-get install block — needed by slack-lists and slack-canvas CLI scripts to parse JSON responses from the Slack API.
2. Added `COPY bin/ /usr/local/bin/` after `COPY agent-runner/ ./` — installs the CLI scripts into PATH so the container agent can call them by name.

## Invariants

- `jq` must be in the existing apt-get block (not a separate RUN layer) to keep the image small
- `COPY bin/` must come after WORKDIR /app but before `RUN npm run build` (ordering relative to other COPY steps)
- The bin/ scripts must be executable at authoring time — COPY preserves permissions
