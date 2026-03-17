# Octos Hub

Community skill hub for [Octos](https://github.com/octos-org/octos). Discover and install skills directly from the CLI.

## Usage

```bash
# Browse all available skills
octos skills search

# Search by keyword
octos skills search slides

# Install a skill package
octos skills install user/repo
```

## Submit Your Skills

Add your skill package by submitting a PR that appends an entry to `registry.json`:

```json
{
  "name": "my-skills",
  "description": "Short description of what your skills do",
  "repo": "your-github-user/your-repo",
  "skills": ["skill-a", "skill-b"],
  "requires": ["git", "node"],
  "tags": ["keyword1", "keyword2"]
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | Package name (unique identifier) |
| `description` | yes | One-line description |
| `repo` | yes | GitHub path `user/repo` |
| `skills` | no | List of individual skill names in the package |
| `requires` | no | External tools needed (e.g. `git`, `node`, `python`) |
| `tags` | no | Searchable keywords |

### Skill Repo Structure

Your repo should contain directories with `SKILL.md` files:

```
your-repo/
  skill-a/
    SKILL.md
    ...
  skill-b/
    SKILL.md
    ...
```

## Auto-Sync

Skill packages can auto-sync their registry entries on CI pass. See [mofa-skills](https://github.com/mofa-org/mofa-skills) for an example using GitHub Actions + the GitHub API.
