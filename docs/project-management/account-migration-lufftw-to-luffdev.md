# Account Migration: lufftw → luffdev

## Task Definition

Migrate all git repositories, environment files, MCP configuration, and Claude Code skills/agents/commands from the `lufftw` Windows user account to the `luffdev` account. Strategy: clone from remote (no local `.git` directory copying).

## Scope

- **Source:** `E:\Developer\lufftw\repo\` (22 git repos, 3 non-git dirs skipped)
- **Target:** `E:\Developer\luffdev\repo\` (repo-workspace already present)
- **GitHub accounts:** `lufftw` (18 repos), `luffdev` (2 repos: organization-data-layer, poi-data-layer-workspace)
- **Components:** git repos, .env files (18 files across 14 repos), MCP config (8 servers), skills (390/project), agents (6), commands (41)

---

## Phase 1: Prepare Source Repos

### 1.1 Add safe.directory entries

Source repos are owned by a different Windows user. For each repo under `E:\Developer\lufftw\repo\`:

```bash
git config --global --add safe.directory "E:/Developer/lufftw/repo/<project>"
```

### 1.2 Commit uncommitted changes

Check all repos with `git status --porcelain`. For any with modifications/untracked files:

```bash
cd E:\Developer\lufftw\repo\<project>
git add -A && git commit -m "chore: stage pending changes for migration"
```

**Known issues encountered:**
- `nul` files are Windows reserved filenames — cannot be added to git (`error: invalid path 'nul'`). Ignore them.
- Empty repos (no commits yet) will fail `git push` with `error: src refspec main does not match any`. Must create an initial commit first.

### 1.3 Add remotes to repos missing them

Check each repo for `git remote get-url origin`. If missing:

```bash
git remote add origin https://github.com/<account>/<project>.git
```

### 1.4 Create missing GitHub repos

If a repo doesn't exist on GitHub yet, create it via GitHub API MCP or manually:

```
mcp__github-api__create_repository(name=<project>, private=true)
```

**Note:** `gh` CLI is not authenticated on this system. Use MCP `github-api` tool or the GitHub web UI.

### 1.5 Push all repos

```bash
git push -u origin <branch>
```

**Priority:** Push repos with unpushed commits first, then the rest.

**Edge cases encountered:**
- `event-crawler` was on branch `merge/batch-crawler-fixes` (not `main`). Push the current branch.
- `poi-data-layer-workspace` remote pointed to `lufftw` but repo only existed under `luffdev`. Update remote URL: `git remote set-url origin https://github.com/luffdev/<project>.git`

### 1.6 Verify

For every repo: `git log origin/<branch>..HEAD` should show 0 commits ahead.

---

## Phase 2: Clone All Repos to Target

```bash
cd E:\Developer\luffdev\repo
git clone https://github.com/<account>/<project>.git
```

- Skip repos that already exist (e.g. `repo-workspace`).
- Use the correct GitHub account per repo (`lufftw` or `luffdev`).
- Verify: 21 repos with `.git` directories, `git config user.name` inherits global `luffdev`.

---

## Phase 3: Environment File Migration

### 3.1 Copy .env files

For each repo with .env files, copy from source to target. Skip `.env.example`, `.env.sample`, `.env.template` variants.

**Repos with .env files (14 repos, 18 files):**

| Repo | Files |
|------|-------|
| event-crawler | `.env` |
| event-image-gen | `.env` |
| event-platform-infra | `.env` |
| event-search-service | `.env`, `backend/.env`, `frontend/.env` |
| gpu-coordinator | `.env` |
| gpu-exporter | `.env` |
| grafana | `.env` |
| image-delivery-service | `.env` |
| mcp-services | `.env` |
| milvus-services | `.env`, `.env.local`, `.env.production` |
| ollama | `.env` |
| poi-data-layer | `.env` |
| prometheus | `.env` |
| taiwan-address-normalizer | `.env` |

### 3.2 Update paths in .env files

Search and replace in all copied `.env*` files:

```
lufftw  → luffdev
luff543 → luffdev
```

**Files that required changes (this migration):**
- `event-search-service/.env` — `PROJECT_PATH=E:\\Developer\\lufftw\\...`
- `mcp-services/.env` — `PROJECT_ROOT=E:\Developer\lufftw\...`

### 3.3 Verify

Scan all `.env*` files for remaining `lufftw` or `luff543` references. Expect zero matches.

---

## Phase 4: MCP Configuration

### 4.1 Global MCP config

`C:\Users\luffdev\.claude.json` — verify `mcpServers` section has all 8 servers and `filesystem` points to `E:\Developer\luffdev\repo`.

**8 servers:** memory, sequential-thinking, git, filesystem, playwright, brave-search, github-api, claude-context.

### 4.2 mcp-services project config

Update path references (`luff543`/`lufftw` → `luffdev`) in:

- `configs/unified-mcp-config.json`
- `configs/claude-core.json`
- `configs/dual-memory-system-config.json`
- `configs/templates/global-mcp-config.json`
- `configs/templates/optimized-mcp-config.template.json`

### 4.3 Project-level .mcp.json

Check all repos for `.mcp.json` files. Verify no stale path references.

---

## Phase 5: Skills Installation

### 5.1 Run agent-skills-mirror batch installer

```bash
node E:\Developer\luffdev\repo\agent-skills-mirror\install-skills.mjs --all E:\Developer\luffdev\repo
```

**Important:** Use the Node.js version (`install-skills.mjs`), NOT the PowerShell version (`install-skills.ps1`). PowerShell scripts with Chinese characters fail due to encoding issues on this system.

This installs 390 skills per project (6 tiers) and syncs `AGENTS.md`.

### 5.2 Install agents and commands

The batch installer (`install-skills.mjs`) only installs skills. Agents and commands must be installed separately from their source repos.

**Sources:**
- **Agents (6):** `alirezarezvani/claude-code-skill-factory` → `.claude/agents/`
- **Commands (41):**
  - `alirezarezvani/claude-code-skill-factory` → `.claude/commands/` (17 commands)
  - `hesreallyhim/awesome-claude-code` → `.claude/commands/evaluate-repository.md` (1)
  - `hesreallyhim/awesome-claude-code` → `resources/slash-commands/*/` → `.claude/commands/` (23 commands)

**Process:**

```bash
TEMP_DIR="/tmp/claude-skills-install"
mkdir -p "$TEMP_DIR"

# Clone sources
git clone --depth 1 https://github.com/alirezarezvani/claude-code-skill-factory.git "$TEMP_DIR/factory"
git clone --depth 1 https://github.com/hesreallyhim/awesome-claude-code.git "$TEMP_DIR/hesreallyhim"

# For each project:
for repo in <all-repos>; do
  dest="E:/Developer/luffdev/repo/$repo/.claude"
  mkdir -p "$dest/agents" "$dest/commands"

  # Agents from factory
  cp -r "$TEMP_DIR/factory/.claude/agents"/* "$dest/agents/"

  # Commands from factory
  cp -r "$TEMP_DIR/factory/.claude/commands"/* "$dest/commands/"

  # Command from hesreallyhim .claude/commands/
  cp "$TEMP_DIR/hesreallyhim/.claude/commands/evaluate-repository.md" "$dest/commands/"

  # Slash commands from hesreallyhim resources/
  for cmd_dir in "$TEMP_DIR/hesreallyhim/resources/slash-commands"/*/; do
    cmd_name=$(basename "$cmd_dir")
    [ ! -d "$dest/commands/$cmd_name" ] && cp -r "$cmd_dir" "$dest/commands/"
  done
done

rm -rf "$TEMP_DIR"
```

**Note:** The bash script `scripts/install-all-skills.sh` in agent-skills-mirror does this for a single project, but it fails on this system due to `declare -A` (associative arrays) + `set -e` incompatibility in Git Bash. Use the manual process above instead.

---

## Phase 6: Git Global Config Cleanup

### 6.1 Remove stale safe.directory entries

```bash
git config --global --unset-all safe.directory
```

### 6.2 Verify final config

```bash
git config --global --list
# Expected:
# user.name=luffdev
# user.email=disklintv1@gmail.com
# init.defaultbranch=main
```

---

## Phase 7: Verification Checklist

### Repos (21)

For every repo under `E:\Developer\luffdev\repo\`:
- [ ] `.git` directory exists
- [ ] `git config user.name` returns `luffdev`
- [ ] Latest commit matches remote HEAD

### .env files (18 files)

- [ ] All .env files present in target repos
- [ ] Zero `lufftw`/`luff543` references in any `.env*` file
- [ ] `PROJECT_PATH`/`PROJECT_ROOT` updated to `luffdev` path

### MCP (8 servers)

- [ ] Global config has all 8 servers
- [ ] `filesystem` server points to `E:\Developer\luffdev\repo`
- [ ] `mcp-services/configs/` has no stale path references
- [ ] Project-level `.mcp.json` files have no stale paths

### Skills / Agents / Commands

Per project:
- [ ] `.claude/skills/` — 390 skills
- [ ] `.claude/agents/` — 6 agents
- [ ] `.claude/commands/` — 41 commands
- [ ] `AGENTS.md` — present and synced

### Git global config

- [ ] No stale `safe.directory` entries
- [ ] `user.name=luffdev`, `user.email=disklintv1@gmail.com`, `init.defaultbranch=main`

---

## Lessons Learned

1. **PowerShell + Chinese encoding:** PowerShell scripts with Chinese characters fail on this system. Always use Node.js alternatives (`install-skills.mjs` over `install-skills.ps1`).
2. **`nul` artifacts:** Windows reserved filename. Cannot be deleted or added to git. Ignore them.
3. **`gh` CLI not authenticated:** Use MCP `github-api` tool or git credential manager for GitHub operations.
4. **Empty repos:** Repos with no commits fail `git push`. Must create an initial commit first.
5. **`install-all-skills.sh` incompatibility:** `declare -A` + `set -e` in Git Bash causes silent failure. Use the manual clone+copy approach for agents/commands.
6. **openskills only installs skills:** The `npx openskills install` command and `install-skills.mjs` only deploy to `.claude/skills/`. Agents (`.claude/agents/`) and commands (`.claude/commands/`) require separate installation from their source repos.
7. **GitHub account split:** Most repos are under `lufftw`, but some were created under `luffdev` when they didn't exist on GitHub. Check remote URLs before cloning.

---

## Complete Repo List (21)

| Repo | GitHub Account | Category |
|------|---------------|----------|
| agent-skills-mirror | lufftw | Main |
| event-crawler | lufftw | Main |
| event-crawler-workspace | lufftw | Workspace |
| event-image-gen | lufftw | Main |
| event-platform-infra | lufftw | Main |
| event-search-service | lufftw | Main |
| event-search-service-workspace | lufftw | Workspace |
| gpu-coordinator | lufftw | Main |
| gpu-exporter | lufftw | Main |
| grafana | lufftw | Main |
| image-delivery-service | lufftw | Main |
| mcp-services | lufftw | Main |
| milvus-services | lufftw | Main |
| ollama | lufftw | Main |
| organization-data-layer | luffdev | Special |
| organization-data-layer-workspace | lufftw | Workspace |
| poi-data-layer | lufftw | Main |
| poi-data-layer-workspace | luffdev | Special |
| prometheus | lufftw | Main |
| repo-workspace | lufftw | Workspace (pre-existing) |
| taiwan-address-normalizer | lufftw | Main |
