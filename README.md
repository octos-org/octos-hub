# Octos Hub

[English](README.md) | [中文](README_CN.md)

Community skill registry for [Octos](https://github.com/octos-org/octos). Browse, install, and publish skills that extend your AI agent with new capabilities.

## Quick Start

```bash
# Browse all available skills
octos skills search

# Search by keyword
octos skills search slides

# Install a skill package (all skills in the repo)
octos skills install mofa-org/mofa-skills

# Install a single skill from a multi-skill repo
octos skills install mofa-org/mofa-skills/mofa-cards

# List installed skills
octos skills list

# Remove a skill
octos skills remove mofa-cards
```

## How Skills Work

A skill is a directory with a `SKILL.md` file that teaches the agent new behaviors. Skills can optionally include executable tools (binaries or scripts) that the agent can call.

```
my-skill/
  SKILL.md            # Required: instructions + metadata
  manifest.json       # Optional: tool definitions + binary distribution
  Cargo.toml          # Optional: Rust source (auto-built on install)
  package.json        # Optional: Node.js (auto-installed on install)
  references/         # Optional: reference docs the agent can read
```

### SKILL.md

Every skill needs a `SKILL.md` with YAML frontmatter:

```markdown
---
name: my-skill
description: Brief description shown in skill listings
version: 1.0.0
author: your-name
always: false
requires_bins: docker,ffmpeg
requires_env: API_KEY
---

# My Skill

Instructions for the agent. Write this as if you're briefing a colleague:
- What this skill does
- When to use it
- Step-by-step usage patterns
- Examples with expected output
```

#### Frontmatter Fields

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `name` | Yes | — | Lowercase identifier with hyphens (e.g., `deep-search`) |
| `description` | Yes | — | One-line description shown in `octos skills list` |
| `version` | No | — | Semver version for update tracking |
| `author` | No | — | Author name or organization |
| `always` | No | `false` | `true` = always included in system prompt. Use sparingly. |
| `requires_bins` | No | — | Comma-separated binaries that must be on PATH |
| `requires_env` | No | — | Comma-separated env vars that must be set |

### manifest.json (for tools)

If your skill provides executable tools, declare them in `manifest.json`:

```json
{
  "name": "my-skill",
  "version": "1.0.0",
  "tools": [
    {
      "name": "my_tool",
      "description": "What this tool does (shown to the LLM)",
      "input_schema": {
        "type": "object",
        "properties": {
          "query": { "type": "string", "description": "Search query" },
          "limit": { "type": "integer", "description": "Max results", "default": 10 }
        },
        "required": ["query"]
      }
    }
  ]
}
```

#### Tool I/O Protocol

Tools receive JSON on **stdin** and must output JSON on **stdout**:

```
stdin:  {"query": "rust async", "limit": 5}
stdout: {"output": "Results here...", "success": true}
```

Exit code 0 = success, non-zero = error. The `output` field is returned to the agent.

#### Tool Executable Resolution

The plugin loader looks for executables in this order:
1. `<skill-dir>/main` — pre-built binary (preferred)
2. `<skill-dir>/<skill-name>` — named binary
3. `<skill-dir>/index.js` — Node.js script (run via `node`)

### Pre-built Binary Distribution

For faster installs (skip compilation), publish platform binaries in your manifest:

```json
{
  "name": "my-skill",
  "version": "1.0.0",
  "timeout_secs": 300,
  "binaries": {
    "darwin-aarch64": {
      "url": "https://github.com/you/repo/releases/download/v1.0.0/my-skill-darwin-aarch64.tar.gz",
      "sha256": "abc123..."
    },
    "darwin-x86_64": {
      "url": "https://github.com/you/repo/releases/download/v1.0.0/my-skill-darwin-x86_64.tar.gz",
      "sha256": "def456..."
    },
    "linux-x86_64": {
      "url": "https://github.com/you/repo/releases/download/v1.0.0/my-skill-linux-x86_64.tar.gz",
      "sha256": "789ghi..."
    }
  },
  "tools": [ ... ]
}
```

**Platform keys**: `darwin-aarch64` (Apple Silicon), `darwin-x86_64` (Intel Mac), `linux-x86_64`, `linux-aarch64`.

The installer downloads the matching binary, verifies the SHA-256 hash, and extracts it. If no binary is available for the platform, it falls back to building from source.

## Writing a Tool in Rust

```rust
// src/main.rs
use serde::{Deserialize, Serialize};
use std::io::Read;

#[derive(Deserialize)]
struct Input {
    query: String,
    #[serde(default = "default_limit")]
    limit: usize,
}
fn default_limit() -> usize { 10 }

#[derive(Serialize)]
struct Output {
    output: String,
    success: bool,
}

fn main() {
    let mut buf = String::new();
    std::io::stdin().read_to_string(&mut buf).unwrap();
    let input: Input = serde_json::from_str(&buf).unwrap();

    let result = format!("Found {} results for '{}'", input.limit, input.query);

    let output = Output { output: result, success: true };
    println!("{}", serde_json::to_string(&output).unwrap());
}
```

Add `serde`, `serde_json` to your `Cargo.toml`. The installer runs `cargo build --release` if no pre-built binary is available.

## Writing a Tool in Node.js

```javascript
// index.js
const input = JSON.parse(require('fs').readFileSync('/dev/stdin', 'utf8'));

const result = `Found results for: ${input.query}`;
console.log(JSON.stringify({ output: result, success: true }));
```

Add a `package.json` with dependencies. The installer runs `npm install` automatically.

## Multi-Skill Repositories

A single repo can contain multiple skills as top-level directories:

```
my-skills/
  skill-a/
    SKILL.md
  skill-b/
    SKILL.md
    manifest.json
    Cargo.toml
    src/main.rs
```

Users can install all skills or pick one:

```bash
octos skills install you/my-skills          # installs skill-a + skill-b
octos skills install you/my-skills/skill-b  # installs only skill-b
```

## Publishing to the Registry

### Step 1: Create your skill repo

Push your skill(s) to a public GitHub repository following the structure above.

### Step 2: Test locally

```bash
# Install from your repo
octos skills install your-user/your-repo

# Verify it works
octos skills list
```

### Step 3: Submit a PR to this repo

Fork [octos-hub](https://github.com/octos-org/octos-hub), add your entry to `registry.json`:

```json
{
  "name": "my-skills",
  "description": "Short description of what your skills do",
  "repo": "your-user/your-repo",
  "skills": ["skill-a", "skill-b"],
  "requires": ["git", "node"],
  "tags": ["keyword1", "keyword2"]
}
```

### Registry Entry Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Unique package name (lowercase, hyphens) |
| `description` | Yes | One-line description (shown in search results) |
| `repo` | Yes | GitHub path `user/repo` |
| `skills` | No | List of individual skill names in the package |
| `requires` | No | External tools needed (e.g., `git`, `node`, `cargo`, `python`) |
| `tags` | No | Searchable keywords (used by `octos skills search`) |

### Step 4: Review

The registry team reviews your PR for:
- Valid `SKILL.md` with required frontmatter
- Working tool executables (if any)
- No malicious code or excessive dependencies
- Accurate description and tags

## Skill Loading

Skills load in priority order (first wins on name conflict):
1. Per-profile skills (`~/.octos/profiles/<profile>/skills/`)
2. Global skills (`~/.octos/skills/`)
3. Built-in skills (cron, skill-store, skill-creator)

Skills with `always: true` are injected into every system prompt. Other skills appear in the skill index — the agent reads them on demand.

## In-Chat Skill Management

Inside an octos gateway chat session:

```
/skills              # list installed skills
/skills install user/repo   # install from GitHub
/skills remove my-skill     # remove a skill
```

The agent can also manage skills programmatically via the built-in `skill-store` tool.

## Best Practices

1. **Keep SKILL.md under 200 lines** — the agent reads it into context
2. **Include concrete examples**, not abstract theory
3. **Use `requires_bins`** to fail fast if external tools are missing
4. **Set `always: true` sparingly** — it adds to every prompt, increasing token usage
5. **Tools should be standalone** — no LLM calls inside tools; let the agent reason
6. **Publish pre-built binaries** for faster installs across platforms
7. **Use SHA-256 verification** for all binary downloads
8. **Version your skills** for update tracking

## Examples

| Package | Repo | Skills | Description |
|---------|------|--------|-------------|
| mofa-skills | [mofa-org/mofa-skills](https://github.com/mofa-org/mofa-skills) | slides, cards, comic, infographic, research, fm, news, github, workflow | AI content creation suite |

## Links

- [Octos](https://github.com/octos-org/octos) — The AI agent framework
- [Skill Creator Guide](https://github.com/octos-org/octos/blob/main/crates/octos-agent/skills/skill-creator/SKILL.md) — Built-in skill for creating new skills
