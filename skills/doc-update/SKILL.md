---
name: doc-update
description: >
  Use when code has changed and vault documentation needs to reflect the
  current codebase state. Analyzes git diff and maps changes to affected
  documents. Requires an existing vault from /init-docs.
argument-hint: "[optional: description of changes, Jira ticket ID, or commit range]"
user-invocable: true
disable-model-invocation: true
---

# /doc-update — Vault Documentation Updater

Update vault documentation after code changes. Analyzes git diff, maps changes to affected vault documents, and updates them.

**Usage:** `/doc-update [optional context]`
- `/doc-update` — analyze last commit, update affected docs
- `/doc-update HN-210: auto-create sales orders from e-shop orders` — use ticket context
- `/doc-update Refaktoroval jsem fakturacni service — rozdelil na tri mensi`
- `/doc-update --range HEAD~5..HEAD` — specific commit range

User context from `$ARGUMENTS` provides change description, ticket ID, or commit range.

**Prerequisite:** Vault must exist with `_Meta/project-profile.yaml`. If not found, stop with error: "Run `/init-docs` first to generate the vault."

## Supporting Files

Read from the `init-docs` skill directory on-demand:

| File | When to read | Purpose |
|------|-------------|---------|
| `${CLAUDE_SKILL_DIR}/../init-docs/references/conventions.md` | Phase 0, Phase 5 | Frontmatter, tags, Mermaid, naming conventions |
| `${CLAUDE_SKILL_DIR}/prompts/update-impact.md` | Phase 2 (large changes only) | Subagent prompt for Explore agents |

## Progress Tracking

Use TaskCreate/TaskUpdate for real-time progress:
- "Phase 0: Loading vault context..."
- "Phase 1: Detecting code changes..."
- "Phase 2: Mapping impact to vault docs..."
- "Phase 3: Auditing affected docs..."
- "Phase 5: Executing updates — 2/4 done..."

Between phases, briefly inform the user of the previous phase's result (1-2 sentences).

## Quality Standards

### Anti-Hallucination Principle

**This skill NEVER speculates.** Every update must be backed by a specific change in the code.

| Situation | Action |
|-----------|--------|
| Fact determinable from code change | Update directly, cite source file and change |
| Cannot determine impact from code | Obsidian callout `[!question]` explaining uncertainty |
| Business reason for the change | NEVER invent — use commit message or `$ARGUMENTS` context only |

### needs-review by Content Reliability

| Reliability | Example | `needs-review` |
|-------------|---------|-----------------|
| **High** — directly from code | "New SalesOrderAutoCreateService added" | `false` |
| **High** — structure/relations | "New dependency: Eshop to SalesOrder" | `false` |
| **Medium** — derived from code | "Service auto-creates sales orders from e-shop orders" | `true` |
| **Low** — motivation/context | "Why auto-create instead of manual workflow" | `true` |

Document-level flag = lowest reliability in the document. If one section is medium, the entire doc gets `needs-review: true`.

### Audience Awareness

Purpose/Účel sections must be readable by non-technical stakeholders (PO, PM, business owners). No class names, no framework terms, no code references — business terminology only. Technical depth belongs in subsequent sections. When updating a Purpose section, preserve this separation.

### Context7 MCP

If context7 MCP is available, use it to verify framework APIs when documenting new code patterns. Same usage as doc-research: `resolve-library-id` + `query-docs`.

## Document Ownership Rules

### Permissions

| Action | doc-update |
|--------|-----------|
| Create new document | No |
| Overwrite `generated: true` + `needs-review: true` | Yes |
| Modify `generated: true` + `needs-review: false` | Yes (with evidence) |
| Modify `generated: false` | **Yes, with per-document approval** |
| Update Mermaid diagrams | Yes |

For `generated: false` documents, require per-document approval in Phase 4.

---

## Phase 0 — Context Loading

**Goal:** Load vault state, project profile, and conventions.

1. **Detect vault:**
   - Path from `$ARGUMENTS` (if specified)
   - `docs/Vault/` (default)
   - `docs/Vault-*/` (glob for project-named vaults)
   - If not found, ask user
2. **Read `{vault}/_Meta/project-profile.yaml`** — modules, tech stack, language
3. **Read `${CLAUDE_SKILL_DIR}/../init-docs/references/conventions.md`**
4. **Sanity check:**
   - Profile exists and has `modules[]`
   - Vault has `00_Home.md`
   - If vault or profile missing, stop with error: "Run `/init-docs` first to generate the vault."

---

## Phase 1 — Change Detection

Determine what code changed. Combine these sources:

```yaml
sources:
  - git diff HEAD~N        # default N=1, or per argument
  - git log --oneline      # commit messages for context
  - $ARGUMENTS             # optional description / Jira ticket ID
```

**If the diff range is unclear, ask:**

> "Analyze changes from:
> a) Last commit (HEAD~1)
> b) Last N commits — how many?
> c) From specific commit/tag
> d) Staged changes (not yet committed)"

**Skip the question if** `$ARGUMENTS` contains an unambiguous range:
- Ticket ID found in `git log` output (grep for it)
- Explicit `--range <range>` argument
- Description matching recent commit messages

**Output:** Structured change list:

```yaml
changedFiles:
  - { path: "src/Entity/Delivery.php", changeType: modified }
  - { path: "src/Services/SalesOrder/AutoCreateService.php", changeType: added }
  - { path: "config/services.yaml", changeType: modified }
commitMessages:
  - "feat(HN-210): auto-create sales orders from e-shop orders"
userContext: "Auto-creating sales orders from e-shop orders"
```

**File classification is tech-agnostic:** Map files through `project-profile.yaml` modules[].path. Files within a module's path belong to that module. Files outside all module paths are classified as architecture/infra.

---

## Phase 2 — Impact Mapping

Map changed files to affected vault documents:

```
Changed file
  -> project-profile.yaml modules[].path
    -> module name
      -> Grep vault for [[Module_Name]] and files with matching domain/ tag
        -> list of affected vault docs
```

Three impact levels:

| Level | Example | Action |
|-------|---------|--------|
| **Direct** | Change in Delivery entity -> `20_Moduly/Dodavky.md` | Update content |
| **Dependency** | New relation Delivery->SalesOrder -> `Mapa_Modulu.md`, `Prodejni_Zakazky.md` | Update relations/diagrams |
| **Infrastructure** | Change in `docker-compose.yml` -> `50_Reference/Technologie.md` | Update reference |

If no module matches changed files, report: "Changed files cannot be mapped to existing vault documents. Consider `/doc-research` for a new topic or `/init-docs` refresh."

### Subagent Dispatch

Only dispatch a subagent if the change is large or cross-cutting:
- Change affects **10+ files**
- Change is a **cross-module refactor**

For most changes (1-5 files), the skill handles analysis directly.

When dispatching:
1. **Read** `${CLAUDE_SKILL_DIR}/prompts/update-impact.md`
2. **Construct** the full agent prompt:
   - Prompt file content
   - `---` separator
   - `## Project Profile` header followed by `project-profile.yaml` content
   - `## Changed Files` header followed by Phase 1 change list
   - `## Vault Documents` header followed by list of vault docs with frontmatter
   - `## User Context` header followed by `$ARGUMENTS`
3. **Dispatch** via Agent tool with `subagent_type: "Explore"`

---

## Phase 3 — Audit

For each affected vault document:

1. **Read** current document content
2. **Read** current state of the relevant source files
3. **Identify** specific discrepancies:
   - Missing new components (added files/classes not mentioned)
   - Removed/renamed files still mentioned in the doc
   - Changed relations/dependencies not reflected
   - Outdated Mermaid diagrams (missing nodes, wrong edges)
   - Inaccurate behavior descriptions

**Output:** Per-document list of specific issues with evidence (source file + what changed).

---

## Phase 4 — Plan (Interactive)

Present the update plan to the user:

```
Code changes (feat HN-210):
  3 files modified, 1 added in modules: SalesOrder, Eshop

Affected vault documents:
1. 20_Moduly/Prodejni_Zakazky.md (generated: true, needs-review: true)
   - ADD: new SalesOrderAutoCreateService to key components
   - ADD: dependency on [[Eshop]] (new relation)
   - UPDATE: diagram — add flow from e-shop order

2. 20_Moduly/Eshop.md (generated: true, needs-review: true)
   - ADD: link to auto-create flow in [[Prodejni_Zakazky]]

3. 10_Architektura/Mapa_Modulu.md (generated: true, needs-review: false)
   - UPDATE: dependency diagram (new edge Eshop -> SalesOrder)

Continue?
```

**Key rules:**
- For `generated: false` documents, require per-document approval (see Document Ownership Rules)
- Wait for user confirmation before proceeding to Phase 5
- If user wants to adjust scope, return to relevant phase

---

## Phase 5 — Execute

Read `${CLAUDE_SKILL_DIR}/../init-docs/references/conventions.md` before executing.

For each approved change:

- **Anti-hallucination:** Update only what can be backed by code changes
- **Flag rules** per content reliability (see Quality Standards)
- **`[!question]` callouts** for areas requiring business context
- **Wikilinks:** Bidirectional per conventions — update both source and target documents
- **Mermaid diagrams:** Updated per conventions (max 50 nodes, ELK for 20+ nodes). Use `language` from profile for node labels.

### Pre-completion Self-check

Before proceeding to Phase 6, verify for each updated document:
- All mentioned file paths still exist (Glob check)
- Version numbers match manifest files (composer.json, package.json, go.mod)
- Entity/class names match actual code identifiers
- Service/file counts are based on Glob results, not estimates

Fix any discrepancies before moving on.

---

## Phase 6 — Validate + Report

1. **Broken links check:**
   - If `obsidianCli: true` in profile: `obsidian unresolved vault=<name> verbose`
   - Otherwise: Grep vault for `[[wikilinks]]` and verify targets exist
2. **Report to user:**
   - What was updated, what requires review
   - Any `[!question]` callouts added
   - If changes fall outside scope of existing vault docs, recommend `/doc-research` for new topic or `/init-docs` refresh
3. **Do NOT commit** — the user commits when ready.

---

## Tech-Agnosticity

Hard rules:

- Skill NEVER hardcodes framework-specific paths or patterns
- All project-specific knowledge comes from `project-profile.yaml` (tech stack, `modules[].path`, language)
- Subagent prompts are parametric — they receive tech stack from profile and choose search strategy themselves
- Vault conventions (`conventions.md`) are tech-agnostic
