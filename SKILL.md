---
name: schematic
description: |
  Reverse engineer a detailed product and technical specification document from a git branch's
  implementation OR an entire project. Use when: (1) a branch has shipped or is in-progress and
  needs documentation, (2) you need to understand what a branch does at product and architecture
  level, (3) onboarding to someone else's feature branch, (4) creating PR descriptions or design
  docs after the fact, (5) user asks to "analyze this branch", "write a spec from the code",
  "document what this branch does", (6) user asks to "analyze this project", "document the whole
  codebase", or "write a spec for this repo". Produces a structured markdown spec covering problem
  statement, product requirements, architecture, technical design, file inventories, testing
  strategy, rollout plan, and risks.
author: Codex
version: 1.2.0
date: 2026-02-17
tags: [documentation, git, branch-analysis, project-analysis, spec, reverse-engineering]
---

# Reverse Engineer Spec from Branch Implementation

## Problem

Feature branches often ship without comprehensive documentation. Entire projects may lack a
coherent product and technical spec. Manually reading every file is slow and error-prone.
This skill systematically extracts a complete spec from either a branch's diff or an entire
project's codebase.

## Context / Trigger Conditions

- User asks to "analyze this branch" or "reverse engineer a spec"
- User asks to "document what this branch does"
- User asks to "analyze this project" or "document the whole codebase"
- User wants a product spec, technical spec, or design doc from existing code
- A branch has many commits and files changed and needs a coherent explanation
- Onboarding to an unfamiliar feature branch or a new project

## Solution

### Phase 0: Mode Selection & Output Naming

Ask the user which analysis mode to use:

- **Branch diff** (default): analyze changes between the current branch and a base branch
- **Full project**: analyze the entire tracked codebase

Determine the output filename:

```bash
# Branch diff mode:
BRANCH=$(git branch --show-current | tr '/' '-')
# Output: docs/${BRANCH}_SPEC.md

# Full project mode:
PROJECT=$(basename "$(git rev-parse --show-toplevel)")
# Output: docs/${PROJECT}_SPEC.md
```

### Phase 1: Scope the Branch (or Project)

Get the full picture before reading any files.

**Branch diff mode:**

```bash
# 1. Identify the base branch (usually main or latest)
git log --oneline <base>..HEAD | head -50

# 2. Get file-level diff stats
git diff --stat <base>...HEAD

# 3. Count the scale
git diff --stat <base>...HEAD | tail -1

# 4. Estimate diff token budget (chars / 4 ≈ tokens)
git diff <base>...HEAD | wc -c
```

**Hitchhiker commit detection (CRITICAL)**: Before proceeding, check whether the branch contains commits from other PRs that were separately merged to the target branch. This is common on un-rebased branches.

```bash
# Get PR commits from GitHub (if a PR exists)
BRANCH_NAME=$(git branch --show-current)
ALL_COMMITS=$(git log --oneline <base>..HEAD | wc -l)
PR_SHAS=$(gh pr list --head "$BRANCH_NAME" --json commits --jq '.[0].commits[].oid' 2>/dev/null)
PR_COMMIT_COUNT=$(echo "$PR_SHAS" | grep -c . 2>/dev/null || echo 0)

# If counts differ, scope to PR-only files
if [ -n "$PR_SHAS" ] && [ "$PR_COMMIT_COUNT" -lt "$ALL_COMMITS" ]; then
  echo "HITCHHIKER COMMITS: $ALL_COMMITS on branch, $PR_COMMIT_COUNT in PR"
  # Get files touched ONLY by PR commits
  PR_FILES=$(for sha in $PR_SHAS; do git diff-tree --no-commit-id --name-only -r "$sha"; done | sort -u)
  # Use: git diff <base>...HEAD -- $PR_FILES (simulates post-rebase diff)
fi
```

When hitchhiker commits are detected, use `git diff <base>...HEAD -- <PR_FILES>` for all subsequent analysis. State this scoping in the output. When not detected, use the full diff.

**Full project mode:**

```bash
# 1. List all tracked files with sizes
git ls-files | head -100

# 2. Count the scale
git ls-files | wc -l

# 3. Estimate total token budget (chars / 4 ≈ tokens)
git ls-files -z | xargs -0 wc -c | tail -1
```
**Agent scaling by token budget:**
- **<50k tokens** → single agent (read all diffs directly)
- **50k–200k tokens** → 2–3 agents
- **200k+ tokens** → 3–4 agents, max ~150k tokens per agent

From the diff stats (scoped if needed), categorize files into groups:
- **Core implementation** (new modules, business logic)
- **Integration points** (modified selectors, reducers, hooks, components)
- **Tests** (unit tests, integration tests, e2e tests)
- **Configuration** (feature flags, env vars, types, configs)
- **Incidental** (formatting, imports, minor refactors)

### Phase 2: Parallel Deep Exploration

Launch 2-4 parallel exploration agents, each focused on a different file group. This is
critical for efficiency — reading 50+ files sequentially is too slow.

**Model allocation:** Use `subagent_type: "Explore"` with `model: "sonnet"` for all
exploration agents. Sonnet handles file reading and analysis (best cost/capability ratio).
The orchestrating model (Opus) plans assignments, synthesizes reports, and infers product
motivation — it should never read files directly.

#### Branch diff mode agents

**Agent 1: Core Implementation**
- All new files (the heart of the feature)
- Focus on: purpose, key types, exported functions, data flow, inter-module connections

**Agent 2: Integration Points**
- Modified selectors, reducers, hooks, components
- Focus on: what changed, why (inferred), how it connects to core implementation

**Agent 3: Tests**
- All test files (unit, integration, e2e)
- Focus on: what behaviors are validated, key assertions, what product requirements they encode

**Agent 4 (if needed): Configuration & Infrastructure**
- Feature flags, env vars, build configs, type declarations
- Focus on: rollout strategy, gating mechanisms, deployment concerns

#### Full project mode agents

Use the same grouping strategy but adjust the focus: instead of "what changed", agents
analyze "what this file/module does and why it exists".

**Agent 1: Core Modules** — primary business logic and domain models
**Agent 2: API / Interface Layer** — entry points, routes, CLI, UI components
**Agent 3: Tests & Config** — test suites, configuration, build setup
**Agent 4 (if needed): Infrastructure** — deployment, CI/CD, scripts, utilities

#### Common agent prompt requirements

Each agent prompt should ask for:
- Purpose of each file
- Key exports and types
- Data flow and dependencies
- How each file connects to others in the group

### Phase 3: Cross-Check for Gaps

After agents return, compare the analyzed files against the full file list:

**Branch diff mode:**

```bash
# List all changed files (language-agnostic)
git diff --name-only <base>...HEAD | sort

# Show small diffs for any files not yet analyzed
git diff <base>...HEAD -- <uncovered-files>
```

**Full project mode:**

```bash
# List all tracked files
git ls-files | sort

# Read any files not yet analyzed
```

Read the remaining files directly. These often contain important details:
- Type declarations (new fields on models)
- Feature flag definitions
- Configuration and environment setup
- Utility functions and shared helpers

### Phase 4: Write the Spec Document

Write to the filename determined in Phase 0. Structure the spec with these sections
(skip sections that don't apply):

```markdown
# [Feature Name / Project Name]
## Reverse-Engineered Product & Technical Specification

## 1. Problem Statement
Why this feature exists. What user/business pain it addresses.
Infer from the nature of the changes and any comments in the code.

## 2. Solution Overview
High-level description of the approach. Key design properties
(transparent, lazy, bounded, etc.).

## 3. Product Requirements
### 3.1 User-Facing Behavior
Table of requirements inferred from tests and UI changes.

### 3.2 Supported Workflows
List of workflows validated by tests.

### 3.3 Scope Boundaries
What is and isn't included.

## 4. Architecture
### 4.1 System Diagram
Mermaid `graph TB` diagram showing component relationships and data flow.
(Mermaid renders natively on GitHub/GitLab — prefer over ASCII.)

### 4.2 Data Lifecycle
Mermaid `sequenceDiagram` or step-by-step description showing flow
from initial state through steady state.

## 5. Technical Design
Subsections for each major design decision:
- Feature flags and gating
- Data models / schema changes
- Key algorithms or patterns
- Integration patterns (how existing code was modified)
- Cache/performance design
- Error handling and fallbacks

## 6. New Files (branch diff mode) / File Inventory (full project mode)
Table: file path, purpose (one line each).

## 7. Modified Files — Key Changes (branch diff mode only)
Table: file path, what changed (one line each).
Include ALL files — even minor ones. The cross-check in Phase 3
catches files that agents missed.
(Skip this section entirely in full project mode.)

## 8. Testing Strategy
### Unit Tests
### Integration / E2E Tests
### Instrumentation / Observability

## 9. Rollout Strategy
How the feature is gated, incremental rollout steps, kill switches.

## 10. Risks and Mitigations
Table: risk, mitigation.

## 11. Summary
Key metrics: files added/modified, lines changed, scope of impact.
```

### Phase 5: Verify Completeness

Cross-check the spec against the source:

**Branch diff mode:**
1. Every file in `git diff --stat` should appear in Section 6 or 7
2. Every test file should be referenced in Section 8
3. Feature flags mentioned in code should appear in Section 5/9
4. The architecture diagram should match the actual data flow discovered by agents

**Full project mode:**
1. Every file in `git ls-files` should appear in Section 6 (File Inventory)
2. Every test file should be referenced in Section 8
3. The architecture diagram should match the actual module structure
4. Entry points and public APIs should be clearly documented

## Verification

- Every file (changed files in branch mode, all tracked files in full project mode) is accounted for in the spec
- The architecture diagram accurately represents the data flow
- Product requirements match what the tests actually validate
- No significant design decisions are missing from the technical design section
- Output file is written to the correct path (`docs/{branch}_SPEC.md` or `docs/{project}_SPEC.md`)

## Example

See the canonical offload spec produced for the `test-parity-mem-exp-99-with-pr16393` branch:
a 12-section document covering 74 changed files across 12 commits, with architecture diagrams,
IndexedDB schema documentation, proxy design details, cache eviction policies, testing strategy
against a real customer dataset, and a complete file inventory.

## Notes

- **Parallel agents are essential**: A branch with 50+ files takes too long to analyze
  sequentially. 3-4 parallel agents cut analysis time by 3-4x.
- **Cross-check is critical**: Agents inevitably miss some files. The Phase 3 cross-check
  catches small but important changes (type declarations, bug fixes, compatibility shims).
- **Infer the "why"**: Code shows "what" but not always "why". Use test assertions, comments,
  commit messages, and the shape of changes to infer product motivation.
- **Output naming convention**:
  - Branch diff → `docs/{branch}_SPEC.md` (e.g., `docs/improve-skill-v1.1_SPEC.md`)
  - Full project → `docs/{project}_SPEC.md` (e.g., `docs/schematic_SPEC.md`)
  - Sanitize branch names: replace `/` with `-` (e.g., `feature/auth` → `feature-auth`)
- **Don't over-document incidentals**: Formatting changes, import reordering, and trailing
  commas can be mentioned in a single line rather than getting their own subsection.
- **Use tables liberally**: File inventories, feature flags, risks — tables are scannable
  and compact.
- **Full project mode** is ideal for: onboarding to a new codebase, creating initial project
  documentation, or when there's no meaningful base branch to diff against.
