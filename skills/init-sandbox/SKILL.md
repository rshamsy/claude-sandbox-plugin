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

### 3. `sandbox.sh`

Do NOT generate this script inline. The canonical copy lives in this skill's base directory at `templates/sandbox.sh`. The script contains bash positional variables (`$0`, `$1`) that slash-command argument substitution blanks out if they are embedded in this markdown file, so always copy the template file directly:

```
cp "<skill-base-directory>/templates/sandbox.sh" ./sandbox.sh
chmod +x sandbox.sh
```

The skill invocation message states the base directory (e.g. "Base directory for this skill: ..."). No placeholder replacement is needed — the script derives PROJECT_NAME from its own directory at runtime via `basename`.


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
    "defaultMode": "bypassPermissions",
    "allow": [],
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
