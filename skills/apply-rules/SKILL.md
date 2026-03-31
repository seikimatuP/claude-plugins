---
name: apply-rules
description: Apply organization-wide rules (from merge-rules output) to the current project. Detects relevant languages/frameworks/integrations from the project's tech stack, merges with existing rules while preserving project-specific .local.md files, and cleans up non-conforming rule files. Use when onboarding a project to org standards, syncing rules after org-wide rule updates, or bootstrapping .claude/rules/ for a new project. Trigger whenever the user mentions applying org rules, syncing shared rules, importing team coding standards, or setting up project rules from a shared source.
allowed-tools: Read, Glob, Grep, Write, Bash(ls *), Bash(mkdir *), Bash(wc *), Bash(mktemp *), Bash(rm -rf /var/folders/*), Bash(rm -rf /tmp/*), Bash(rm .claude/*), Bash(gh api *), Bash(gh auth status *)
---

# Apply Rules

Applies organization-wide rules (produced by merge-rules) to the current project. Detects the project's tech stack, selects relevant rules, merges them with existing extract-rules output, and ensures the final structure conforms to the extract-rules/merge-rules convention.

## Usage

```text
/apply-rules <source>                  # Apply from GitHub URL or local path
/apply-rules                           # Apply using config file
/apply-rules --config <path>           # Apply using specified config file
/apply-rules --dry-run                 # Show what would change without writing
/apply-rules --dry-run <source>        # Dry run with specified source
```

`<source>` points directly to the rules directory (merge-rules or extract-rules output):
- GitHub: `https://github.com/org/repo/tree/main/.claude/rules`
- Local: `~/org-rules/.claude/rules`

## Configuration

Config file search order:
1. `--config <path>` argument
2. `.claude/apply-rules.local.md` (project-level)
3. `~/.claude/apply-rules.local.md` (user-level)

**File format:** YAML frontmatter only (no markdown body), same convention as `extract-rules.local.md` and `merge-rules.local.md`.

```yaml
---
# Rules directory (GitHub URL or local path)
# GitHub: https://github.com/org/repo/tree/main/.claude/rules
# Local: ~/org-rules/.claude/rules
source: https://github.com/org/repo/tree/main/.claude/rules

# Alternative: specify GitHub source components separately
# (useful when branch name contains "/" or for tags/SHAs)
# source_repo: org/repo
# source_ref: feature/rules-v2
# source_path: .claude/rules

# Output directory in target project (default: .claude/rules/)
output_dir: .claude/rules/

# Auto-detect which rules to apply (default: true)
# When false, applies ALL rules from source
auto_detect: true

# Explicitly include rules even if not auto-detected
include: []
# Example: [languages/typescript, integrations/rails-inertia]

# Explicitly exclude rules even if auto-detected
exclude: []
# Example: [frameworks/rails-views]

# Report language (default: ja)
language: ja
---
```

CLI argument `<source>` overrides the config's `source` field.

## Processing Flow

### Step 1: Load Configuration

1. If CLI `<source>` argument provided, use it (overrides config `source`)
2. Search for config file (see search order above)
3. Parse YAML frontmatter, apply defaults for omitted fields
   - **`language` resolution order:** Skill config → Claude Code settings (`~/.claude/settings.json` → `language` field) → default `ja`
4. Validate:
   - `source` must be specified (via CLI argument or config file)
   - If neither: Error "No source specified. Provide a GitHub URL or local path as argument, or create `.claude/apply-rules.local.md` with a `source:` field."

### Step 2: Fetch Source Rules

**If source is a local path:**

1. Expand `~` and resolve to absolute path
2. Verify directory exists and contains rule files (`.md`)
3. Read directly from this location

**If source is a GitHub URL:**

Parse URL to extract owner, repo, branch, and path:
- `https://github.com/{owner}/{repo}/tree/{branch}/{path}`
- Example: `https://github.com/org/repo/tree/main/.claude/rules`
  → owner: `org`, repo: `repo`, branch: `main`, path: `.claude/rules`

**Note on ambiguous refs:** Branch names may contain `/` (e.g., `feature/rules-v2`), and refs can also be tags or SHAs. Simple URL splitting cannot reliably separate ref from path. To handle this robustly:
- Try resolving ref candidates from longest prefix first using `gh api repos/{owner}/{repo}/git/ref/{candidate}`
- Alternatively, the user can specify components separately in the config:
  ```yaml
  source_repo: org/repo
  source_ref: feature/rules-v2
  source_path: .claude/rules
  ```
  When these fields are present, they take precedence over URL parsing.
- For the common case (branch = `main` or `master`), simple URL parsing works.

Fetch using `gh api`:

1. Verify authentication: `gh auth status`
2. Create temp directory: `mktemp -d`
3. List top-level directory contents:
   ```
   gh api repos/{owner}/{repo}/contents/{path}?ref={branch}
   ```
   For each entry with `type: "dir"`, recursively fetch subdirectory contents:
   ```
   gh api repos/{owner}/{repo}/contents/{path}/{subdir}?ref={branch}
   ```
   This dynamically discovers all categories (not limited to `languages/`, `frameworks/`, `integrations/`).
4. For each `.md` file found, fetch content and decode:
   ```
   gh api repos/{owner}/{repo}/contents/{file_path}?ref={branch} --jq '.content | @base64d'
   ```
   Save to tmpdir preserving directory structure
5. Temp dir is cleaned up in Step 8

**Inventory source files:**
- Glob for `**/*.md` and `**/*.examples.md` under the source rules directory
- Skip `project.md` and `project.examples.md` (inherently project-specific)
- Skip `.local.md` files (merge-rules output should not contain these, but handle gracefully)
- Parse each file: extract YAML frontmatter (`paths:`) and body sections

### Step 3: Detect Target Project Tech Stack

Analyze the current working directory to determine which source rules are relevant. Detection is best-effort, based on dependency files (e.g., `Gemfile`, `package.json`) and project directory structure (e.g., `app/controllers/`).

Read `references/detection-heuristics.md` for the full detection table mapping indicators to rule files. If the source contains rule files not covered in the table, use AI judgment to match them against the project's dependencies and file structure.

**Apply overrides:**
- Add `include:` entries to the detected set
- Remove `exclude:` entries from the detected set
- If `auto_detect: false`: start with ALL source rules, then apply `exclude` only

### Step 4: Filter and Propose

**If `auto_detect: false`:** Apply ALL source rules (minus `exclude` entries). Skip the proposal step entirely — no integration proposals or skipped rules. Proceed directly to Step 5.

**If `auto_detect: true` (default):**

1. **Auto-matched rules**: Rules that match detected tech stack → apply automatically
2. **Integration proposals**: For integrations NOT detected in the project but related to a detected framework (e.g., source has `integrations/rails-pundit` but project doesn't use `pundit`), present them to the user:

   > The following integration rules are available in the source but were not detected in your project. Would you like to apply any of them?
   > - `integrations/rails-pundit` — Authorization library Pundit rules
   > - `integrations/rails-good-job` — Job queue GoodJob rules
   >
   > These are useful if you plan to adopt these libraries or want to familiarize with org coding standards in advance.

   Apply only those the user approves.

3. **Skipped rules**: Rules for tech not detected and not in related frameworks → skip, list in report

### Step 5: Inventory Existing Target Rules

1. Check if `{output_dir}/` exists in the target project
2. If exists, read all files: `.md`, `.local.md`, `.examples.md`
3. Parse frontmatter and body sections for each
4. Categorize files the same way as source files
5. **Detect hybrid format**: If any `.md` file contains `## Project-specific patterns` (hybrid format from `split_output: false`), note this. The merge step will convert hybrid to split format because the split format (separate `.md` and `.local.md`) is the standard expected by both extract-rules and merge-rules, and mixing formats causes confusion when rules flow back through the pipeline

### Step 5.5: Normalize File Names

Before merging, align target file names with source (canonical) names. This prevents duplicate files for the same concept (e.g., `rails-controller.md` vs `rails-controllers.md`).

1. Compare target file names against source file names within the same category
2. Detect naming variants: singular/plural (`controller`/`controllers`), minor differences for the same concept
3. If a target file matches a source file by content/`paths:` overlap but has a different name, rename the target to match the source (canonical) name
4. Report all renames in the summary

### Step 6: Merge Rules

For each filtered source rule file, determine the merge action:

**6a. Merge `.md` files (Principles):**

**Case: No existing `.md`**
1. Copy source `.md`
2. If source contains `## Project-specific patterns` (promoted patterns from merge-rules):
   - Move that section to a new `.local.md` file instead
   - The `.md` file keeps only `## Principles` and `## Examples` reference

**Case: Existing `.md` exists**
1. **`paths:` frontmatter**: Union of all path patterns, deduplicate
2. **Hybrid → split conversion**: If existing `.md` contains `## Project-specific patterns` (hybrid format):
   - Extract that section and move to `.local.md` (create if not exists, append if exists)
   - Remove the section from `.md`, keeping only `## Principles`
3. **`## Principles`**:
   - Match principles by name (text before parenthetical hints)
   - Source principle not in target → Add
   - Target principle not in source → Keep (project may have added its own)
   - Same principle, same meaning but different hints → Union hints from both
   - Same principle name but different content → Present both versions to the user and ask which to adopt:
     > The following principle differs between org rules and project rules. Which should be used?
     > - **Org:** `Immutability (spread, map/filter/reduce, const)`
     > - **Project:** `Immutability (freeze, deep clone, readonly)`
     >
     > 1. Adopt org rule
     > 2. Keep project rule
     > 3. Keep both
4. **`## Project-specific patterns` from source**:
   Promoted patterns from org rules belong in `.local.md`, not `.md`, because `.md` is reserved for portable principles. Keeping this boundary clean ensures that when this project's rules flow back into merge-rules, portable and local content don't get mixed.
   - If target has a `.local.md` → Append promoted patterns not already present
   - If no target `.local.md` → Create new `.local.md` with promoted patterns
   - Never put promoted patterns into the target `.md`

**6b. Preserve `.local.md` files:**

`.local.md` files contain project-specific patterns discovered by extract-rules — conventions unique to this codebase that would be lost if overwritten by org rules. Preserving them ensures the project retains its domain knowledge while adopting shared standards.

- Existing `.local.md` content is never modified or overwritten
- Only append: promoted patterns from source that don't already exist
- **Duplicate detection**: Match patterns by inline code signature (backtick portion before ` - `). Use AI judgment for semantic equivalence (e.g., `useAuth()` and `useAuth() → { user, login, logout }` refer to the same pattern). Same logic as merge-rules Step 5

**6c. Merge `.examples.md` files:**

**Case: No existing `.examples.md`**
- Copy source `.examples.md`
- Remove `## Project-specific Examples` unless corresponding promoted patterns were added to `.local.md`

**Case: Existing `.examples.md` exists**
1. **`## Principles Examples`**: Add examples from source for principles not already covered in target
2. **`## Project-specific Examples`**: Preserve ALL existing entries (they correspond to existing `.local.md` patterns). Only ADD examples for newly promoted patterns from 6a. Never remove existing project-specific examples

**6d. Ensure `## Examples` reference:**
- Every `.md` and `.local.md` that has a corresponding `.examples.md` must end with:
  ```markdown
  ## Examples
  When in doubt: ./<name>.examples.md
  ```

### Step 7: Structure Conformance Check + Auto-cleanup

Scan `output_dir` for files that don't conform to extract-rules/merge-rules convention:

**Valid patterns:**
- `{category}/{name}.md`
- `{category}/{name}.local.md`
- `{category}/{name}.examples.md`
- `project.md`
- `project.examples.md`

**Valid categories:** `languages/`, `frameworks/`, `integrations/`

**Non-conforming file handling:**
1. Read the non-conforming file and analyze its content
2. Determine the appropriate conforming file(s) to migrate rules into (based on category, content, and `paths:` hints)
3. Present the migration plan to the user for confirmation:
   > The following non-conforming files were detected. Migrate their rules to conforming files and delete them?
   > - `frameworks/old-custom-rules.md` → migrate to `frameworks/rails.md`
   > - `ruby-rules.md` → migrate to `languages/ruby.md`
4. User approval → merge rules into conforming file(s) and delete non-conforming files
5. Report all migrations and deletions

**Note:** `project.*` files (e.g., `project.rules.md`) require extra caution — always confirm individually with the user before migrating or deleting.

### Step 8: Cleanup

If source was a GitHub URL, remove the temp directory: `rm -rf <tmpdir>`

### Step 9: Report Summary

Display report. Report headers are always in English, content in the configured `language`.

```
# Apply Rules Report

## Source
- https://github.com/org/repo/tree/main/.claude/rules (15 rule files)

## Target Project Detection
- Languages: ruby
- Frameworks: rails, rails-controllers, rails-models, rails-views
- Integrations: rails-devise, rails-pundit

## Applied Rules
| File | Action | Principles | Promoted → .local.md |
|------|--------|------------|----------------------|
| languages/ruby.md | Merged | +3 added, 7 kept | 1 promoted |
| frameworks/rails.md | Created | 13 | 0 |
| frameworks/rails-controllers.md | Created | 5 | 0 |
| integrations/rails-devise.md | Merged | +1 added | 0 |

## Preserved (untouched)
- languages/ruby.local.md (project-specific)
- frameworks/rails.local.md (project-specific)

## User-approved Integrations
- integrations/rails-pundit (not detected, approved by user)

## Skipped (not relevant to project)
- languages/typescript.md
- frameworks/react.md
- frameworks/nextjs.md
- integrations/rails-stripe.md

## Structure Cleanup
- frameworks/old-custom.md → rules migrated to frameworks/rails.md → deleted

## Conflicts (resolved by user)
- languages/ruby.md: "Immutability" → user chose: Keep both
```

## Conflict Handling

Summary of user-confirmation points and automatic actions:

| Situation | Action |
|-----------|--------|
| Principle in source, not in target | Auto-add |
| Principle in target, not in source | Auto-keep |
| Same principle, different hints | Auto-union hints |
| Same principle name, different content | **Ask user**: adopt org / keep project / keep both |
| Promoted pattern not in target `.local.md` | Auto-append (duplicate check by inline code signature) |
| Promoted pattern already in `.local.md` | Auto-skip |
| Non-conforming file detected | **Ask user**: confirm migration and deletion |
| `project.*` non-conforming file | **Ask user**: confirm individually |
| Undetected integration rule (related framework) | **Ask user**: propose for approval |
| Existing `.local.md` content | Never modified (append-only for promoted patterns) |
| Existing `.examples.md` project-specific examples | Never removed (append-only for new promoted pattern examples) |
