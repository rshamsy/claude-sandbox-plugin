---
name: init-sandbox
description: Initialize a Docker-based Claude Code sandbox for the current project with permission profiles, container tracking, and session resume support
disable-model-invocation: true
---

# Initialize Claude Sandbox

Create all sandbox infrastructure files for the current project. Derive PROJECT_NAME from the current directory name.

## Files to Create

Create ALL of the following files immediately:

### 1. `Dockerfile.claude-sandbox`

```dockerfile
FROM python:3.11-slim

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    jq \
    && rm -rf /var/lib/apt/lists/*

# Install Node.js (required for Claude Code)
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y nodejs

# Install Claude Code CLI
RUN npm install -g @anthropic-ai/claude-code

# Create workspace directory
WORKDIR /workspace

# Set up non-root user for safety
RUN useradd -m -s /bin/bash claude && chown -R claude:claude /workspace

# Create .claude directory structure for plugins
RUN mkdir -p /home/claude/.claude/plugins/cache/claude-plugins-official/ralph-loop/latest \
    && mkdir -p /home/claude/.claude/plugins/marketplaces \
    && chown -R claude:claude /home/claude/.claude

USER claude

# Copy plugin configuration
COPY --chown=claude:claude .claude-plugins/ /home/claude/.claude/plugins/
COPY --chown=claude:claude .claude-plugins/settings.json /home/claude/.claude/settings.json

CMD ["bash"]
```

### 2. `docker-compose.sandbox.yml`

Replace PROJECT_NAME with the actual current directory name:

```yaml
services:
  claude-sandbox:
    build:
      context: .
      dockerfile: Dockerfile.claude-sandbox
    container_name: PROJECT_NAME-sandbox
    volumes:
      - .:/workspace
      - ~/.claude:/home/claude/.claude
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - CLAUDE_CODE_SKIP_ONBOARDING=1
    working_dir: /workspace
    stdin_open: true
    tty: true
    deploy:
      resources:
        limits:
          memory: 4G
          cpus: '2'
```

### 3. `sandbox.sh` (chmod +x after creation)

Replace PROJECT_NAME with the actual current directory name:

```bash
#!/bin/bash
# Launch Claude Code in Docker sandbox
#
# Usage:
#   ./sandbox.sh              - New session in safe mode
#   ./sandbox.sh full         - New session in full trust mode
#   ./sandbox.sh shell        - Open container shell
#   ./sandbox.sh resume       - Resume: pick container + session interactively (full trust)
#   ./sandbox.sh resume safe  - Resume: pick container + session interactively (safe mode)

MODE=${1:-"safe"}
TRUST_MODE=${2:-"full"}
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
STATE_FILE="${SCRIPT_DIR}/.sandbox-state.json"

# Derive project name from directory
PROJECT_NAME="$(basename "$SCRIPT_DIR")"

# --- Helper: record container signature ---
record_container() {
    local name="$1"
    local id
    id=$(docker inspect --format '{{.Id}}' "$name" 2>/dev/null | head -c 12)
    local image
    image=$(docker inspect --format '{{.Config.Image}}' "$name" 2>/dev/null)
    local created
    created=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

    # Read existing state or start fresh
    local entries="[]"
    if [ -f "$STATE_FILE" ]; then
        entries=$(jq '.containers // []' "$STATE_FILE" 2>/dev/null || echo "[]")
    fi

    # Remove stale entry with same name, then append
    entries=$(echo "$entries" | jq --arg n "$name" '[.[] | select(.name != $n)]')
    local new_entry
    new_entry=$(jq -n \
        --arg name "$name" \
        --arg id "$id" \
        --arg image "$image" \
        --arg created "$created" \
        '{name: $name, id: $id, image: $image, created_at: $created}')
    entries=$(echo "$entries" | jq --argjson e "$new_entry" '. + [$e]')

    jq -n --argjson c "$entries" '{containers: $c}' > "$STATE_FILE"
    echo "Container recorded in .sandbox-state.json"
}

# --- Helper: pick a container interactively ---
pick_container() {
    local containers=()

    # First, try containers from the state file (known to this project)
    if [ -f "$STATE_FILE" ]; then
        local known_names
        known_names=$(jq -r '.containers[].name' "$STATE_FILE" 2>/dev/null)
        while IFS= read -r name; do
            [ -z "$name" ] && continue
            local status
            status=$(docker ps -a --filter "name=^${name}$" --format '{{.Status}}' 2>/dev/null)
            if [ -n "$status" ]; then
                containers+=("${name}\t${status}")
            fi
        done <<< "$known_names"
    fi

    # Fallback: scan docker for containers matching the project name
    if [ ${#containers[@]} -eq 0 ]; then
        mapfile -t containers < <(docker ps -a --filter "name=${PROJECT_NAME}" --format '{{.Names}}\t{{.Status}}' 2>/dev/null)
    fi

    if [ ${#containers[@]} -eq 0 ]; then
        echo "Error: No containers found. Run './sandbox.sh full' first." >&2
        exit 1
    fi

    if [ ${#containers[@]} -eq 1 ]; then
        SELECTED=$(echo -e "${containers[0]}" | cut -f1)
        echo "Auto-selected container: $SELECTED" >&2
    else
        echo "" >&2
        echo "Available containers:" >&2
        for i in "${!containers[@]}"; do
            local name=$(echo -e "${containers[$i]}" | cut -f1)
            local status=$(echo -e "${containers[$i]}" | cut -f2)
            echo "  $((i+1))) $name  [$status]" >&2
        done
        echo "" >&2
        read -rp "Select container [1-${#containers[@]}]: " choice
        if ! [[ "$choice" =~ ^[0-9]+$ ]] || [ "$choice" -lt 1 ] || [ "$choice" -gt ${#containers[@]} ]; then
            echo "Invalid selection." >&2
            exit 1
        fi
        SELECTED=$(echo -e "${containers[$((choice-1))]}" | cut -f1)
    fi

    echo "$SELECTED"
}

# --- Helper: ensure container is running ---
ensure_running() {
    local container="$1"
    if ! docker ps --format '{{.Names}}' | grep -q "^${container}$"; then
        echo "Starting stopped container: $container"
        docker start "$container"
    fi
}

# --- Helper: update Claude Code to the latest version ---
# Runs as root because the CLI is installed in the root-owned npm global dir.
update_claude() {
    local container="$1"
    echo "Updating Claude Code to the latest version..."
    if ! docker exec -u root "$container" npm install -g @anthropic-ai/claude-code@latest; then
        echo "WARNING: Claude Code update failed (offline?). Continuing with installed version." >&2
    fi
}

# --- Helper: apply permission profile ---
apply_profile() {
    local mode="$1"
    mkdir -p .claude/profiles
    if [ "$mode" = "full" ]; then
        cp .claude/profiles/full-trust.json .claude/settings.local.json 2>/dev/null || true
    else
        cp .claude/profiles/safe-mode.json .claude/settings.local.json 2>/dev/null || true
    fi
}

# --- Resume mode ---
if [ "$MODE" = "resume" ]; then
    CONTAINER_NAME=$(pick_container)

    if [ "$TRUST_MODE" = "safe" ]; then
        echo "Resuming in safe mode..."
        CLAUDE_CMD="claude --resume"
        apply_profile "safe"
    else
        echo "Resuming in full trust mode..."
        CLAUDE_CMD="claude --dangerously-skip-permissions --resume"
        apply_profile "full"
    fi

    ensure_running "$CONTAINER_NAME"
    update_claude "$CONTAINER_NAME"
    docker exec -it "$CONTAINER_NAME" $CLAUDE_CMD
    exit 0
fi

# --- Normal modes (full / safe / shell) ---
CONTAINER_NAME="${PROJECT_NAME}-sandbox"

if [ "$MODE" = "full" ]; then
    echo "WARNING: Running in full trust mode - all commands allowed"
    CLAUDE_CMD="claude --dangerously-skip-permissions"
    apply_profile "full"
elif [ "$MODE" = "shell" ]; then
    echo "Opening container shell (run 'claude' to start Claude Code)"
    CLAUDE_CMD="bash"
    apply_profile "safe"
else
    echo "Running in safe mode with restricted permissions"
    CLAUDE_CMD="claude"
    apply_profile "safe"
fi

# Check if container exists
if docker ps -a --format '{{.Names}}' | grep -q "^${CONTAINER_NAME}$"; then
    ensure_running "$CONTAINER_NAME"
    update_claude "$CONTAINER_NAME"
    echo "Attaching to container..."
    docker exec -it "$CONTAINER_NAME" $CLAUDE_CMD
else
    # Container doesn't exist - create it detached, update Claude, then attach
    echo "Creating new container..."
    docker compose -f docker-compose.sandbox.yml build
    docker compose -f docker-compose.sandbox.yml up -d claude-sandbox
    update_claude "$CONTAINER_NAME"

    # Record the new container's signature
    record_container "$CONTAINER_NAME"

    docker exec -it "$CONTAINER_NAME" $CLAUDE_CMD
fi
```

### 4. `.claude/profiles/safe-mode.json`

```json
{
  "permissions": {
    "allow": [
      "Read", "Edit", "Write", "Glob", "Grep",
      "Bash(python:*)", "Bash(pip:*)", "Bash(pytest:*)",
      "Bash(git status:*)", "Bash(git diff:*)", "Bash(git log:*)",
      "Bash(git add:*)", "Bash(git commit:*)",
      "Bash(ls:*)", "Bash(tree:*)", "Bash(mkdir:*)",
      "Bash(npm:*)", "Bash(node:*)",
      "WebSearch", "WebFetch"
    ],
    "deny": ["Bash(rm -rf:*)", "Bash(sudo:*)"]
  },
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### 5. `.claude/profiles/full-trust.json`

```json
{
  "permissions": {
    "allow": ["*"],
    "deny": []
  },
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### 6. `.claude-plugins/settings.json`

```json
{
  "enabledPlugins": {
    "ralph-loop@claude-plugins-official": true
  }
}
```

### 7. `.claude-plugins/installed_plugins.json`

```json
{
  "version": 2,
  "plugins": {
    "ralph-loop@claude-plugins-official": [
      {
        "scope": "user",
        "installPath": "/home/claude/.claude/plugins/cache/claude-plugins-official/ralph-loop/latest",
        "version": "latest",
        "installedAt": "2026-01-01T00:00:00.000Z"
      }
    ]
  }
}
```

### 8. `.claude-plugins/config.json`

```json
{"autoUpdate": true}
```

### 9. `.claude-plugins/known_marketplaces.json`

```json
{
  "claude-plugins-official": {
    "url": "https://github.com/anthropics/claude-plugins-official",
    "trusted": true
  }
}
```

### 10. Create `.dockerignore`

```
.env
```

### 11. Append to `.gitignore`

```
# Claude sandbox
.claude/settings.local.json
.sandbox-state.json
.env
```

## After Creation

1. Run: `chmod +x sandbox.sh`
2. Display this summary:

```
Sandbox initialized!

Files created:
  - Dockerfile.claude-sandbox
  - docker-compose.sandbox.yml
  - sandbox.sh
  - .dockerignore
  - .claude/profiles/safe-mode.json
  - .claude/profiles/full-trust.json
  - .claude-plugins/ (plugin config)

To start:
  1. Set ANTHROPIC_API_KEY in your environment
  2. ./sandbox.sh        (safe mode)
     ./sandbox.sh full   (full trust)
     ./sandbox.sh shell  (bash shell)
     ./sandbox.sh resume [full|safe]  (resume container + session)

Container tracking:
  - .sandbox-state.json records container name/ID after first run
  - ./sandbox.sh resume reads this file to find your containers
```
