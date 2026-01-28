---
name: AI Primitives Sharing Solutions
overview: Multiple conceptual approaches for facilitating discovery, sharing, and reuse of Cursor AI primitives (rules, prompts, skills) across the Mural engineering team with seamless developer experience.
todos: []
---

# AI Primitives Sharing Solutions

## Current State Analysis

Your team currently has:

- **Global rules** in `~/.cursor/rules/` (commit standards, security, gh-cli workarounds)
- **Global skills** in `~/.cursor/skills/` (kubectl commands, PR workflows, API testing)
- **Repo-specific rules** varying by project:
  - `mural-api`: 7 rules (TypeScript standards, MDK workflow, API-worker patterns)
  - `murally`: 15 rules (React patterns, accessibility, testing, styling)
- **YAML frontmatter** with metadata (description, globs, triggers, alwaysApply flags)

**Key insights:**

- Mix of global and repo-specific content
- Rich metadata already exists in frontmatter
- Engineers have diverse needs across different repos
- Some content is universal (commit style), some is project-specific (React patterns)

---

## Critical Design Challenge: Three-Tier Rule Governance

### The Complexity

Your environment has three distinct tiers of AI primitives, each serving different purposes:

**Tier 1: Committed Repo Rules (Official)**

- Location: `repo/.cursor/rules/*.mdc` (in version control)
- Purpose: Project standards, used by CI/CD, team-wide enforcement
- Stability: High - changes go through code review
- Audience: All developers + automation (CI agents)
- Examples: `mural-api/.cursor/rules/002-project-structure.mdc`, `murally/.cursor/rules/react-component-patterns.mdc`
- Governance: Managed via standard Git workflow (PR review, CODEOWNERS)

**Tier 2: Personal Experimental Rules (Developer Sandbox)**

- Location: `~/.cursor/rules/*.mdc` OR `repo/.cursor/rules/*.mdc` (gitignored)
- Purpose: Personal preferences, experimentation, workflow optimization
- Stability: Low - rapid iteration, individual tweaks
- Audience: Individual developer only
- Examples: Personal debugging tricks, preferred code style variations, custom AI behavior tweaks
- Governance: None - developer autonomy

**Tier 3: Shared Community Rules (The Missing Layer)**

- Location: **This is what we're designing**
- Purpose: Crowdsourced patterns, emerging best practices, cross-repo utilities
- Stability: Medium - tested by individuals, not yet "official"
- Audience: Opt-in by interested developers
- Examples: Experimental React patterns, new testing approaches, AI prompt experiments
- Governance: **This is the product decision to make**

### The Maturation Path

Rules naturally evolve through lifecycle stages:

```
Personal Experiment → Community Shared → Committed Official
(Tier 2)           → (Tier 3)         → (Tier 1)

Or they might stay at any level indefinitely:
- Some personal rules never need sharing (individual quirks)
- Some community rules never become official (useful but niche)
- Some committed rules might be extracted back to community (deprecation)
```

### Key Product Questions Each Solution Must Answer

1. **Discovery Separation:** How do developers distinguish between community rules vs official repo rules?

2. **Installation Conflicts:** What happens if a community rule conflicts with a committed rule? (Same filename, different content)

3. **Promotion Workflow:** How does a community rule graduate to committed repo rule? (Copy/paste? Reference? Automated PR?)

4. **Demotion Workflow:** When a committed rule is deprecated, should it become a community rule for legacy users?

5. **CI Transparency:** How does CI know to use only Tier 1 rules and ignore Tier 2/3? (Critical for reproducibility)

6. **Namespace Collision:** How to prevent `react-patterns.mdc` in community library from shadowing `react-patterns.mdc` in committed repo?

7. **Update Semantics:** If a developer has community rule installed and repo commits an official version, what happens?

8. **Opt-in vs Opt-out:** Are shared community rules something developers pull explicitly, or do they auto-install for repos?

---

## Solution 1: Git-Based Catalog with Smart CLI

**Architecture:** Dedicated Git repository as single source of truth + CLI tool for discovery and sync

### Core Components

**Shared Library Repository Structure:**

```
mural-cursor-library/
├── rules/
│   ├── global/
│   │   ├── commit-message-style.mdc
│   │   └── secrets-security.mdc
│   ├── backend/
│   │   ├── typescript-standards.mdc
│   │   └── api-worker-patterns.mdc
│   └── frontend/
│       ├── react-component-patterns.mdc
│       └── accessibility-standards.mdc
├── skills/
│   ├── kubectl-commands.skill.md
│   └── pr-workflows.skill.md
├── catalog.json  # Metadata index for fast search
└── README.md
```

**CLI Tool (`mural-cursor` or `mcursor`):**

```bash
# Discovery
mcursor search "react patterns"
mcursor list --category frontend
mcursor show rules/frontend/react-component-patterns.mdc

# Installation
mcursor install rules/frontend/react-component-patterns.mdc
mcursor install rules/backend/* --repo mural-api

# Publishing
mcursor publish ~/.cursor/rules/new-pattern.mdc --category backend
mcursor publish .cursor/rules/* --as-global false

# Updates
mcursor pull  # Update all installed items
mcursor diff rules/frontend/react-component-patterns.mdc  # Compare local vs published

# Management
mcursor list --installed
mcursor unlink rules/frontend/react-component-patterns.mdc
```

### Developer Experience Flow

1. **Bootstrap:** `mcursor init` creates config in `~/.mcursor/config.json` tracking installed items
2. **Discover:** Browse via CLI or visit GitHub repo directly
3. **Install:** CLI symlinks or copies file to appropriate location (`~/.cursor/` or `repo/.cursor/`)
4. **Modify:** Edit local copy as normal
5. **Publish:** CLI detects changes, prompts for update/new publish, opens PR to catalog repo
6. **Sync:** Periodically `mcursor pull` to get updates from shared library

### Key Features

- **Symlink vs Copy mode:** Symlinks for global rules (auto-update), copies for customized versions
- **Version tracking:** Git commits/tags provide versioning
- **Diff before update:** Show changes before accepting updates
- **Namespace collision detection:** Warn if installing file that conflicts with existing
- **Metadata enrichment:** CLI adds installation metadata to frontmatter

### Pros

- Leverages familiar Git workflows
- Full version history via Git
- No central infrastructure needed
- Easy to review changes via PRs
- Can evolve with existing Git-based code review culture

### Cons

- Requires CLI installation and maintenance
- Potential sync conflicts if developers customize heavily
- No real-time collaboration features
- Requires discipline to publish back changes

### Three-Tier Governance Analysis

**How it handles the distinction:**

**Tier 1 (Committed Repo Rules):**

- Remain in standard Git workflow, untouched by CLI
- CLI could read `repo/.cursor/rules/` to detect conflicts before installing community rules
- Developers manually promote by copying community rule content into repo and committing via standard PR

**Tier 2 (Personal Experimental):**

- Lives in `~/.cursor/rules/` or `repo/.cursor/rules/` (gitignored)
- CLI's `mcursor publish` command extracts these to community library
- Developer controls what gets published via explicit commands

**Tier 3 (Community Shared):**

- Lives in dedicated `mural-cursor-library` Git repo
- CLI installs to `~/.cursor/rules/` by default (global personal space)
- Alternative: `mcursor install --repo-local` installs to `repo/.cursor/rules/` but adds to `.gitignore`

**Maturation workflow:**

```bash
# Personal → Community
mcursor publish ~/.cursor/rules/experimental-pattern.mdc --category experimental

# Community → Committed (manual process)
# 1. Developer reviews community rule effectiveness
# 2. Manually copies to repo's committed .cursor/rules/
# 3. Updates frontmatter (remove publishToTeam flag)
# 4. Creates PR to repo via standard Git workflow
# 5. Team reviews, merges
# 6. Optionally: mcursor unpublish experimental-pattern (deprecate community version)
```

**Conflict handling:**

- CLI warns: "⚠️  `react-patterns.mdc` exists in committed repo rules. Installing to `~/.cursor/rules/` instead. Files will load in this order: [committed repo] → [global]. Consider if this creates conflicts."
- Advanced: CLI could analyze glob patterns in frontmatter to detect semantic conflicts

**CI transparency:**

- CI only sees committed `repo/.cursor/rules/` (no ~/.cursor mounted)
- GitHub Actions, Docker builds, etc. work unchanged
- Community rules never affect CI - by design

**Key strengths:**

- Clear separation: committed stays in repo, community lives elsewhere
- Flexible installation targets (global vs repo-local)
- Simple promotion path (manual copy preserves review culture)

**Key limitations:**

- Manual promotion is friction (no automated "promote to official" button)
- Namespace collisions require developer awareness
- No automatic deprecation when community rule becomes official

---

## Solution 2: NPM-Style Package Manager

**Architecture:** Treat cursor files as packages with dependency management and semantic versioning

### Core Concept

Model after NPM ecosystem where:

- Each "package" is a collection of related rules/skills/prompts
- Packages have dependencies (e.g., `react-patterns` depends on `typescript-standards`)
- Semantic versioning for breaking changes
- Lock file for reproducible environments

### Package Structure

**Example package.json:**

```json
{
  "name": "@mural-cursor/react-patterns",
  "version": "2.1.0",
  "description": "React component patterns and best practices",
  "category": "frontend",
  "files": [
    "react-component-patterns.mdc",
    "accessibility-standards.mdc"
  ],
  "dependencies": {
    "@mural-cursor/typescript-standards": "^1.5.0",
    "@mural-cursor/testing-standards": "^2.0.0"
  },
  "keywords": ["react", "components", "patterns"],
  "author": "frontend-team",
  "license": "INTERNAL"
}
```

### CLI Commands

```bash
# Package registry operations
mcursor init  # Creates package.json for your cursor config
mcursor search react
mcursor info @mural-cursor/react-patterns

# Installation
mcursor add @mural-cursor/react-patterns
mcursor add @mural-cursor/backend-standards --save-dev
mcursor add @mural-cursor/api-patterns@1.2.0  # Specific version

# Update management
mcursor outdated  # Show available updates
mcursor update @mural-cursor/react-patterns
mcursor update --all

# Publishing
cd my-cursor-rules
mcursor publish  # Reads local package.json
mcursor version major|minor|patch
```

### Pros

- Familiar to JS developers
- Automatic dependency resolution
- Clear versioning semantics
- Lock file ensures reproducibility

### Cons

- More complex infrastructure (registry server)
- Overkill for simple rule sharing
- Dependency hell potential
- Higher maintenance burden

---

## Solution 3: Monorepo with Git Submodules

**Architecture:** Single shared repo included as submodule in each project

### Structure

```
# In each project repo
project-repo/
├── .cursor/
│   ├── rules/           # Project-specific rules (committed)
│   └── shared/          # Git submodule → mural-cursor-shared
│       ├── rules/
│       └── skills/
```

### Setup

```bash
# Add shared library as submodule
git submodule add git@github.com:tactivos/mural-cursor-shared.git .cursor/shared

# Update to latest
git submodule update --remote
```

### Pros

- Native Git integration
- Version pinning per repo
- No additional tooling needed

### Cons

- Submodule complexity (notorious for confusion)
- All-or-nothing updates
- No selective installation

---

## Solution 4: Cursor Settings Sync + Shared Workspace

**Architecture:** Leverage Cursor's built-in settings sync with team workspace

### Approach

Use Cursor's settings sync feature combined with:
- Shared workspace configuration
- `.cursor/settings.json` with team presets
- Cursor's native extension/rule discovery (if available)

### Pros

- Zero additional tooling
- Native Cursor experience
- Automatic sync

### Cons

- Limited to Cursor's sync capabilities
- Less granular control
- May not support all use cases

---

## Recommendation: Phased Approach

### Phase 1: Simple Git Catalog (Now)

Start with Solution 1 (Git-Based Catalog) in its simplest form:

1. Create `tactivos/mural-cursor-library` repo
2. Organize existing rules/skills with clear categories
3. Document manual installation process in README
4. No CLI yet - just git clone/copy

**Why:** Validates the content catalog without tooling investment

### Phase 2: Basic CLI (1-2 months)

Build minimal CLI with:
- `mcursor install <path>` - Copy file to ~/.cursor/
- `mcursor list` - Show available items
- `mcursor search <term>` - Search metadata

**Why:** Reduces friction once catalog proves valuable

### Phase 3: Advanced Features (3-6 months)

Based on usage patterns, add:
- Conflict detection
- Update notifications
- Publishing workflow
- Symlink mode

**Why:** Only build complexity when justified by adoption

---

## Implementation Checklist

### Immediate Actions

- [ ] Create `tactivos/mural-cursor-library` repository
- [ ] Define category structure (global, backend, frontend, devops)
- [ ] Migrate existing global rules from ~/.cursor/rules/
- [ ] Write README with manual installation instructions
- [ ] Share with team for feedback

### Short-term (CLI MVP)

- [ ] Scaffold CLI project (TypeScript, commander.js)
- [ ] Implement `install`, `list`, `search` commands
- [ ] Add to Homebrew tap or npm for easy distribution
- [ ] Document CLI usage

### Medium-term (Polish)

- [ ] Add conflict detection
- [ ] Implement symlink mode
- [ ] Build update checker
- [ ] Create VS Code extension for discovery (optional)
