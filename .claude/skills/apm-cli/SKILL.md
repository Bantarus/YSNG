---
name: apm-cli
description: >
  Complete CLI reference for APM (Agent Package Manager) -- all commands, flags, and
  workflows for installing, compiling, packing, auditing, and managing AI agent
  dependencies. Use whenever running APM commands, managing dependencies, troubleshooting
  installation issues, setting up CI/CD with APM, or working with lockfiles. Also use
  when the user mentions apm install, apm compile, apm pack, apm audit, apm deps,
  dependency management, or lockfile operations.
---

# APM CLI Reference

Complete operational reference for the APM command-line interface.

## Quick Reference

```bash
apm init [NAME] [--yes] [--plugin]     # scaffold new project
apm install [PKGS...] [OPTIONS]        # install dependencies + deploy primitives
apm uninstall PKGS...                  # remove packages
apm compile [OPTIONS]                  # compile instructions into AGENTS.md/CLAUDE.md
apm pack [OPTIONS]                     # create portable bundle
apm unpack BUNDLE [OPTIONS]            # extract bundle
apm audit [PACKAGE] [OPTIONS]          # scan for hidden Unicode characters
apm deps list|tree|info|clean|update   # manage dependencies
apm view PACKAGE [versions]            # show package metadata or remote refs
apm outdated                           # check for stale dependencies
apm mcp list|search|show               # browse MCP registry
apm run SCRIPT [--param k=v]           # execute workflow (experimental)
apm update [--check]                   # update APM CLI itself
```

## Core Workflow

The typical APM workflow is:

```bash
apm init my-project         # 1. scaffold
cd my-project
# edit .apm/ primitives     # 2. add content
apm install owner/package   # 3. add dependencies
apm install                 # 4. install all + deploy
apm compile                 # 5. optional: compile for Codex/Gemini
```

## apm init

Scaffold a new APM project. Creates a minimal `apm.yml`.

```bash
apm init                          # interactive, current dir
apm init --yes                    # auto-detect defaults, current dir
apm init my-project               # create new directory
apm init my-plugin --plugin       # plugin authoring mode
```

`--plugin` creates both `plugin.json` and `apm.yml` with `devDependencies`.
Plugin names: kebab-case, `^[a-z][a-z0-9-]{0,63}$`, max 64 chars.

Auto-detected: name (from dir), author (from git config), version (1.0.0).

## apm install

Install dependencies from `apm.yml` and deploy the project's own `.apm/` content.

```bash
apm install                                    # install all from apm.yml
apm install owner/repo                         # install specific package
apm install owner/repo#v1.0.0                  # pinned version
apm install ComposioHQ/awesome-claude-skills/brand-guidelines  # virtual package
apm install https://gitlab.com/acme/rules.git  # full URL
apm install ./packages/my-skills               # local path
apm install NAME@MARKETPLACE                   # from marketplace
```

### Key Flags

| Flag | Description |
|------|-------------|
| `--target [copilot\|claude\|cursor\|codex\|opencode\|all]` | Force deployment target |
| `--update` | Re-resolve all refs to latest commits |
| `--force` | Overwrite locally-authored files; bypass security scan blocks |
| `--dry-run` | Preview without installing |
| `--only [apm\|mcp]` | Install only one dependency type |
| `--dev` | Add to devDependencies (excluded from plugin bundles) |
| `-g, --global` | Install to user scope (~/.apm/) |
| `--verbose` | Show individual file paths and error details |
| `--parallel-downloads N` | Concurrent downloads (default: 4) |
| `--trust-transitive-mcp` | Trust self-defined MCP servers from transitive deps |

### What install Does

1. Downloads packages to `apm_modules/`
2. Deploys primitives from packages AND your local `.apm/` to target dirs
3. Auto-detects targets: `.github/`, `.claude/`, `.cursor/`, `.opencode/`
4. Writes/updates `apm.lock.yaml`
5. Scans for hidden Unicode characters (blocks on critical findings)

### Auto-Bootstrap

- **With packages + no apm.yml**: auto-creates minimal apm.yml
- **Without packages + no apm.yml**: suggests `apm init`

### Target Auto-Detection

| Project has | Targets deployed |
|-------------|-----------------|
| `.github/` only | Copilot/VSCode |
| `.claude/` only | Claude |
| Both | All detected targets |
| Neither | `.github/` created as fallback |

### Diff-Aware Installation

- Packages removed from apm.yml are cleaned up on next `apm install`
- Ref/version changes trigger re-download automatically
- MCP config changes are re-applied without `--force`

## apm uninstall

```bash
apm uninstall owner/repo                    # by shorthand
apm uninstall https://github.com/owner/repo.git  # by URL (same effect)
apm uninstall -g owner/repo                 # from user scope
apm uninstall --dry-run owner/repo          # preview
```

Removes: package from apm.yml, apm_modules/ folder, all deployed files (tracked in
lockfile), orphaned transitive deps.

## apm compile

Compile instructions into AGENTS.md and/or CLAUDE.md. Optional for Copilot, Claude,
and Cursor users (they read native primitives). Needed for Codex, OpenCode
(instructions), and Gemini.

```bash
apm compile                         # auto-detect target
apm compile --target claude         # force Claude output
apm compile --target all            # both AGENTS.md and CLAUDE.md
apm compile --dry-run               # preview placement decisions
apm compile --verbose               # show optimization analysis
apm compile --watch                 # auto-recompile on changes
apm compile --validate              # check primitives without writing
apm compile --clean                 # remove orphaned AGENTS.md files
apm compile --local-only            # ignore dependencies
```

### Target Output

| Target | Output Files |
|--------|-------------|
| `copilot` / `vscode` / `agents` | AGENTS.md |
| `claude` | CLAUDE.md |
| `codex` | AGENTS.md |
| `all` | Both |

### Compilation Config (in apm.yml)

```yaml
compilation:
  target: all
  strategy: distributed         # or single-file
  output: AGENTS.md
  resolve_links: true
  source_attribution: true
  exclude:
    - "apm_modules/**"
    - "tmp/**"
  placement:
    min_instructions_per_file: 1
```

### Distributed Strategy

Default mode. Creates multiple AGENTS.md files across the directory tree based on
`applyTo` patterns. Minimizes context pollution -- each file only sees relevant
instructions.

### Default Exclusions (always applied)

`node_modules`, `__pycache__`, `.git`, `dist`, `build`, `apm_modules`, hidden dirs.

## apm pack

Create a portable bundle from installed dependencies.

```bash
apm pack                            # default: apm format, auto-target
apm pack --archive                  # .tar.gz archive
apm pack --target copilot           # only .github/ files
apm pack --target claude            # only .claude/ files (with cross-target remapping)
apm pack --format plugin            # standalone plugin directory
apm pack -o dist/                   # custom output directory
apm pack --dry-run                  # preview
```

### Prerequisites

1. `apm.lock.yaml` must exist (run `apm install` first)
2. All deployed files must exist on disk
3. No local path dependencies (replace with remote refs first)

### Plugin Format

`apm pack --format plugin` remaps `.apm/` content into plugin-native paths:

| APM source | Plugin output |
|---|---|
| `.apm/agents/*.agent.md` | `agents/*.agent.md` |
| `.apm/skills/*/SKILL.md` | `skills/*/SKILL.md` |
| `.apm/prompts/*.prompt.md` | `commands/*.md` |
| `.apm/instructions/*.instructions.md` | `instructions/*.instructions.md` |
| `.apm/hooks/*.json` | `hooks.json` (merged) |

Generates `plugin.json`. Excludes `devDependencies`.

### Cross-Target Path Mapping

When packing for a different target than installed, skills/ and agents/ paths are
automatically remapped. Commands, instructions, and hooks are target-specific
and NOT remapped.

## apm unpack

Extract a bundle into a project.

```bash
apm unpack bundle.tar.gz                 # extract to current dir
apm unpack bundle.tar.gz -o /path/to/    # extract to specific dir
apm unpack bundle.tar.gz --skip-verify   # skip completeness check
apm unpack bundle.tar.gz --dry-run       # preview
apm unpack bundle.tar.gz --force         # deploy despite Unicode findings
```

Behavior: additive-only (never deletes), bundle overwrites on conflict,
lockfile NOT copied to output.

## apm audit

Scan for hidden Unicode characters that could embed invisible instructions.

```bash
apm audit                               # scan all installed packages
apm audit owner/repo                    # scan specific package
apm audit --file .cursorrules           # scan arbitrary file
apm audit --strip                       # remove dangerous characters
apm audit --strip --dry-run             # preview strip
apm audit -f sarif -o report.sarif      # SARIF output for CI
apm audit -f markdown                   # GitHub step summary
apm audit --ci                          # lockfile consistency gate
apm audit --ci --policy org             # with org policy checks
apm audit --ci --no-fail-fast           # run all checks even after failure
```

### Exit Codes (content scan)

| Code | Meaning |
|------|---------|
| 0 | Clean |
| 1 | Critical findings (tag chars, bidi overrides) |
| 2 | Warnings only (zero-width chars, bidi marks) |

### Exit Codes (--ci mode)

| Code | Meaning |
|------|---------|
| 0 | All checks passed |
| 1 | One or more checks failed |

## apm deps

Manage dependencies.

```bash
apm deps list                    # table of installed packages with counts
apm deps list -g                 # user-scope packages
apm deps list --all              # both project and user scope
apm deps tree                    # hierarchical dependency tree
apm deps info PACKAGE            # alias for apm view
apm deps clean [--yes]           # remove all apm_modules/
apm deps update [PKGS...]        # re-resolve and update
apm deps update --verbose        # detailed update info
apm deps update -g               # update user-scope deps
```

## apm view

```bash
apm view owner/repo              # installed package metadata
apm view package-name            # short-name lookup
apm view owner/repo versions     # list remote tags/branches (no clone needed)
apm view -g owner/repo           # user scope
```

## apm outdated

```bash
apm outdated                     # check all locked deps
apm outdated -g                  # user-scope deps
apm outdated --verbose           # show available tags
apm outdated -j 8                # parallel checks
```

## apm mcp

Browse the MCP server registry.

```bash
apm mcp list [--limit N]         # list available servers
apm mcp search QUERY [--limit N] # search servers
apm mcp show SERVER_NAME         # show server details
```

## apm marketplace

Manage plugin marketplaces (GitHub repos with marketplace.json).

```bash
apm marketplace add OWNER/REPO [--name NAME] [--branch BRANCH]
apm marketplace list
apm marketplace browse NAME
apm marketplace update [NAME]
apm marketplace remove NAME [--yes]
apm search "QUERY@MARKETPLACE" [--limit N]
apm install NAME@MARKETPLACE
```

## apm run (Experimental)

Execute prompt scripts via AI runtimes.

```bash
apm run SCRIPT_NAME --param key=value
apm preview SCRIPT_NAME --param key=value    # preview without executing
apm list                                      # show available scripts
```

Runtimes: `copilot` (recommended), `codex`, `llm`.

```bash
apm runtime setup codex
apm runtime list
apm runtime status
apm runtime remove codex
```

## apm update

Update APM CLI itself.

```bash
apm update --check     # check for updates
apm update             # update to latest
```

Manual: `curl -sSL https://aka.ms/apm-unix | sh` (Linux/macOS)

## Lockfile (apm.lock.yaml)

The lockfile records exact resolved state. Commit it to version control.

### Structure

```yaml
lockfile_version: "1"
generated_at: "2026-03-09T14:00:00Z"
apm_version: "0.7.7"
dependencies:
  - repo_url: https://github.com/owner/repo
    resolved_commit: a1b2c3d4...       # full 40-char SHA
    resolved_ref: v2.1.0               # tag/branch that resolved
    version: "2.1.0"                   # from package's apm.yml
    depth: 1                           # 1=direct, 2+=transitive
    package_type: apm_package
    content_hash: "sha256:..."         # file tree hash
    deployed_files:                    # every file APM placed
      - .github/instructions/security.instructions.md
      - .github/agents/security-auditor.agent.md
mcp_servers:
  - io.github.github/github-mcp-server
```

### Key Fields per Dependency

| Field | Description |
|-------|-------------|
| `repo_url` | Source repo URL (or `_local/<name>` for local) |
| `resolved_commit` | Exact commit SHA (remote only) |
| `depth` | 1=direct, 2+=transitive |
| `resolved_by` | Parent dependency (transitive only) |
| `is_dev` | true for devDependencies |
| `deployed_files` | All files APM placed for this dep |
| `content_hash` | SHA-256 of package file tree |
| `source` | "local" for local path deps |

### Lifecycle

| Event | Effect |
|-------|--------|
| `apm install` (first) | Created |
| `apm install` (subsequent) | Read, locked commits reused |
| `apm install --update` | Re-resolved, overwritten |
| `apm deps update` | Re-resolved |
| `apm pack` | Enriched copy in bundle (project lockfile unchanged) |
| `apm uninstall` | Entries removed |

### Rules

- MUST NOT be manually edited
- SHOULD be committed to version control
- Sorted by depth (ascending), then repo_url (lexicographic)
- All paths use forward slashes (POSIX format)

## Global (User-Scope) Installation

```bash
apm install -g owner/repo       # install globally
apm uninstall -g owner/repo     # remove globally
apm deps list -g                # list global packages
```

| Item | Project scope | User scope (-g) |
|------|--------------|-----------------|
| Manifest | `./apm.yml` | `~/.apm/apm.yml` |
| Modules | `./apm_modules/` | `~/.apm/apm_modules/` |
| Deployed | `./.github/` etc. | `~/.copilot/`, `~/.claude/` etc. |

Restrictions: no local path deps at user scope, no MCP at user scope.

## CI/CD Integration

### GitHub Actions

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: microsoft/apm-action@v1
      - run: apm install
      - run: apm audit --ci
      - run: apm pack --archive
      - uses: actions/upload-artifact@v4
        with:
          name: apm-bundle
          path: build/*.tar.gz

  consume:
    needs: setup
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with: { name: apm-bundle, path: ./bundle }
      - run: tar xzf ./bundle/*.tar.gz -C .
```

### apm-action Modes

```yaml
# Install mode (default)
- uses: microsoft/apm-action@v1

# Pack mode
- uses: microsoft/apm-action@v1
  with: { pack: true }

# Restore mode (no APM needed)
- uses: microsoft/apm-action@v1
  with: { bundle: ./path/to/bundle.tar.gz }
```

## Authentication

### GitHub (recommended: fine-grained token)

```bash
export GITHUB_CLI_PAT=your_fine_grained_token
# Permissions: Contents: Read, Metadata: Read
```

### Other Hosts

APM delegates to git for non-GitHub hosts:
- Public repos: work via HTTPS without auth
- Private via SSH: configure SSH keys
- Private via HTTPS: configure git credential helper

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "Authentication failed" | Check `GITHUB_CLI_PAT` or `GITHUB_TOKEN` env var |
| "Package validation failed" | Verify repo has `.apm/` dir or `apm.yml` or `SKILL.md` |
| "Circular dependency" | Review and remove circular refs in dependency chain |
| "File conflicts" | Use `--verbose` to see which files, `--force` to overwrite |
| "apm.lock.yaml not found" (pack) | Run `apm install` first |
| "deployed files missing" (pack) | Run `apm install` to restore |
| Empty bundle | Check `--target` matches where files were deployed |
| Skill name validation | Must be lowercase, 1-64 chars, hyphens only |
