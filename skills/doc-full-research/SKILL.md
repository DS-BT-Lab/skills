---
name: doc-full-research
description: >
  Use when you need comprehensive documentation coverage across all project
  modules, or when multiple areas need deep research in sequence.
  Requires an existing vault from /init-docs.
argument-hint: "[focus guidance] [--autonomous] [--policy skip-human|auto-all|preserve-human] [--prompt-file path]"
user-invocable: true
disable-model-invocation: true
---

# /doc-full-research — Full-Project Deep Research Orchestrator

Orchestrate deep research across an entire project. Decomposes the codebase into scoped research units, then dispatches sequential subagents that each run the doc-research workflow with a fresh context window.

**Usage:** `/doc-full-research [optional context]`
- `/doc-full-research` — interactive mode, research all modules
- `/doc-full-research "Focus on state machines and workflows"` — with focus guidance
- `/doc-full-research --autonomous` — fully autonomous (all approvals upfront)
- `/doc-full-research --autonomous --policy auto-all` — autonomous, approve all including human docs
- `/doc-full-research --autonomous --prompt-file path/to/custom.md` — autonomous with custom research template

**Prerequisite:** Vault must exist with `_Meta/project-profile.yaml` and `00_Home.md`. If not found, stop with error: "Run `/init-docs` first to generate the vault."

**Core principle:** Context isolation. Each subagent gets a fresh, full context window to deeply analyze one area. Sequential execution ensures each subagent sees previous subagents' vault updates.

## Supporting Files

Read at runtime. Phase 1 files are read once and cached for all subagent prompts.

| File | When to read | Purpose |
|------|-------------|---------|
| `${CLAUDE_SKILL_DIR}/../doc-research/SKILL.md` | Phase 1.1 (once, cache) | Embedded into every subagent prompt |
| `${CLAUDE_SKILL_DIR}/../doc-research/prompts/research-analysis.md` | Phase 1.1 (once, cache) | Embedded for broad topic analysis |
| `${CLAUDE_SKILL_DIR}/../init-docs/references/conventions.md` | Phase 1.1 (once, cache) | Embedded for writing standards |
| `${CLAUDE_SKILL_DIR}/../init-docs/references/diataxis-guide.md` | Phase 1.1 (once, cache) | Embedded for doc classification |
| `${CLAUDE_SKILL_DIR}/../init-docs/templates/module.md` | Phase 1.1 (once, cache) | Embedded for new module doc creation |
| `${CLAUDE_SKILL_DIR}/../init-docs/templates/adr.md` | Phase 1.1 (once, cache) | Embedded if ADR needed |
| `${CLAUDE_SKILL_DIR}/../init-docs/templates/howto.md` | Phase 1.1 (once, cache) | Embedded if howto needed |
| `${CLAUDE_SKILL_DIR}/prompts/scope-decomposition.md` | Phase 0.3 | Grouping heuristics reference |
| `{vault}/_Meta/project-profile.yaml` | Phase 0.1 | Project structure source of truth |

## Progress Tracking

Use TaskCreate/TaskUpdate for real-time progress:
- "Phase 0: Loading vault context..."
- "Phase 0.2: Assessing vault state ({N} module docs found)..."
- "Phase 0.3: Building dependency graph and grouping modules..."
- "Phase 1: Research unit {name} ({i}/{N})..."
- "Phase 2: Validating vault consistency..."

Between phases, briefly inform the user of the previous phase's result (1-2 sentences).

---

## Setup for Autonomous Mode

For autonomous operation (CI/CD, scheduled runs), configure permissions in `.claude/settings.local.json` BEFORE running:

```json
{
  "permissions": {
    "allow": [
      "Read(docs/Vault*/**)",
      "Write(docs/Vault*/**)",
      "Edit(docs/Vault*/**)",
      "Glob",
      "Grep"
    ]
  }
}
```

Note: Plugin-installed skill files are readable by default. Only vault and codebase access need explicit permissions.

On startup, if `--autonomous` is set, verify required tool access. If permissions are insufficient, warn:
"Autonomous mode requires pre-configured permissions. See 'Setup for Autonomous Mode' section."

---

## Phase 0 — Setup & Scope Decomposition

### 0.0 First Run vs Re-run Detection

Check for `{vault}/_Meta/.full-research-state.yaml`.
- **Missing** — first run: interactive Phase 0 (steps 0.1-0.6)
- **Present** — re-run: read state file, load `approvalPolicy`, `focusGuidance`, `promptFile`, `unitsCompleted`, `permanentlyFailed`. Skip to 0.2 (vault re-assessment). No user interaction.

### 0.1 Argument Parsing & Context Loading

1. Parse `$ARGUMENTS` for flags: `--autonomous`, `--policy <value>`, `--prompt-file <path>`, remaining free text as focus guidance
2. Detect vault: from `$ARGUMENTS` path -> `docs/Vault/` -> `docs/Vault-*/` glob
3. Read `{vault}/_Meta/project-profile.yaml`
4. If `--prompt-file` specified: read and validate the file exists
5. Verify prerequisite: vault has `00_Home.md`. If not -> error: "Run `/init-docs` first."
6. Ensure `_Meta/.full-research-state.yaml` is in vault's `.gitignore` (read `.gitignore`, append if missing)

### 0.2 Vault State Assessment

Scan the **entire vault** — not just the modules directory. List all vault top-level directories and assess every `.md` document in each section.

For each `.md` file across all vault sections:
- Read YAML frontmatter: `generated`, `needs-review`, `status`
- Count content lines (excluding frontmatter)
- Check for Mermaid diagram blocks
- Check for `[!question]` callouts
- Classify depth using criteria from `${CLAUDE_SKILL_DIR}/prompts/scope-decomposition.md` Section 7:
  - **shallow** — <30 content lines, no diagrams
  - **moderate** — 30-100 lines, some detail
  - **deep** — 100+ lines, diagrams, detailed sections
- Deep docs -> SKIP by default
- On re-run: compare with `unitsCompleted` from state file, focus on what's still not deep

**Vault section discovery:** List vault top-level directories. Each directory represents a documentation section (architecture, modules, decisions, guides, reference, etc.). All sections are research candidates — the skill covers the entire vault, not just modules.

**Non-module sections** (architecture, reference, etc.) become additional research units appended AFTER module units in the queue. Their topics are formulated from the section name and existing doc titles. They are dispatched to doc-research the same way as module units — no special handling needed.

### 0.3 Module Grouping & Dependency Ordering

1. Read `${CLAUDE_SKILL_DIR}/prompts/scope-decomposition.md`
2. Build dependency graph: for each module from `project-profile.yaml`, grep source files for import/use patterns (Section 1 of scope-decomposition.md, selected by tech stack). Build adjacency map.
3. Topological sort (Section 2): order modules leaf-first (foundation modules first, complex dependents last)
4. Group into research units (Section 3): domain coupling -> type matching -> functional proximity -> size balancing
5. Apply sizing thresholds (Section 4)
6. Append non-module vault sections (architecture, reference, guides, etc.) as additional research units after all module units — they reference module docs and should be researched last

### 0.4 Boundary Context Generation

For each research unit, generate (using templates from scope-decomposition.md Section 5-6):
- **Topic:** Natural language description for doc-research invocation
- **Focus scope:** Module list with paths from profile
- **Boundary context:** Adjacent modules from dependency map (reference only)
- **Already-documented dependencies:** Vault doc paths of modules processed in earlier units or already deep

**Non-module section context generation:**
- Topic: formulated from section name + titles of existing docs in that section (e.g., "Deep-dive into architecture section — system overview, module map, integrations, patterns")
- Boundary context: all completed module units (their vault doc paths) as already-documented
- Dispatched to doc-research using the same subagent prompt as module units — doc-research handles any topic type

### 0.5 Approval Policy (first run only)

Present policy options to user:

```
Approval policy for autonomous research:

1. Documents with `generated: true` -> auto-approve all changes
2. Documents with `generated: false` -> (choose one):
   a) Skip entirely (safest, recommended)
   b) Enrich but preserve existing content
   c) Full rewrite (destructive)
3. New documents (modules without vault docs) -> auto-create with `generated: true`

This policy will be saved and reused on subsequent runs.
```

If `--autonomous` and no prior state file: use default policy (`skip-human`) without asking.

Save chosen policy to state file.

### 0.6 Present Plan to User (first run only)

On re-runs, skip — auto-scope to remaining shallow modules and proceed.

Template:

```
Full-project deep research plan:

Project: {profile.projectContext} ({N} modules, vault language: {profile.language})

Research queue ({M} units, {K} skipped as deep):

{For each unit:}
N. {SKIP if deep | RESEARCH} {unit name}: {module list}
   Context: {boundary modules from dependency map}
   Docs: {vault doc paths with depth classification}

Options:
a) Run all ({M} research sessions)
b) Select specific units
c) Adjust grouping
d) Force re-research on skipped units

After approval, research proceeds autonomously — no further interaction needed.
```

If `--autonomous`: skip presentation, proceed with all non-deep units.

---

## Phase 1 — Sequential Research Dispatch

### 1.1 Subagent Prompt Construction (Pre-injection)

Subagents do NOT invoke the Skill tool. The orchestrator reads and embeds the doc-research workflow directly into each subagent's prompt.

**Read once and cache** (at start of Phase 1):
1. `${CLAUDE_SKILL_DIR}/../doc-research/SKILL.md` — full content, unmodified
2. `${CLAUDE_SKILL_DIR}/../doc-research/prompts/research-analysis.md`
3. `${CLAUDE_SKILL_DIR}/../init-docs/references/conventions.md`
4. `${CLAUDE_SKILL_DIR}/../init-docs/references/diataxis-guide.md`
5. `${CLAUDE_SKILL_DIR}/../init-docs/templates/module.md`
6. `${CLAUDE_SKILL_DIR}/../init-docs/templates/adr.md`
7. `${CLAUDE_SKILL_DIR}/../init-docs/templates/howto.md`
8. If `--prompt-file` was specified: custom prompt file content

**Per-unit prompt assembly:**

```
You are a documentation research agent running in an automated pipeline.
Work AUTONOMOUSLY — do not ask questions or wait for approval.

## Your Task
Research topic: "{generated topic from Phase 0.4} --autonomous --policy {policy}"
Scope: {boundary context from Phase 0.4}

## Already-Documented Dependencies
{For each boundary module already processed by previous subagents:
 - Read its vault doc, include path + first 5 lines as summary context}

## Vault Location
{vault path}

## Project Profile
{project-profile.yaml content}

---

## doc-research Workflow
{Full content of doc-research SKILL.md — UNMODIFIED}

## Reference: Research Analysis Prompt
{Content of research-analysis.md}

## Reference: Documentation Conventions
{Content of conventions.md}

## Reference: Diataxis Guide
{Content of diataxis-guide.md}

## Template: Module Document
{Content of module.md template}

## Template: ADR Document
{Content of adr.md template}

## Template: How-to Guide
{Content of howto.md template}

{If --prompt-file was provided:}
## Custom Research Instructions
{Content of custom prompt file}

---

## When Complete
Provide a brief summary:
- Documents created/enriched (with paths)
- [!question] callouts added
- generated:false documents skipped/preserved
- Any issues encountered
```

### 1.2 Sequential Execution

```
For each unit in approved_units (ordered by dependency depth):
    TaskUpdate: "Phase 1: Research {unit.name} ({i}/{N})..."

    result = Agent(
        subagent_type: "general-purpose",
        prompt: constructed_prompt(unit),
        description: "Research: {unit.name}"
    )

    Record result summary (docs created/enriched, issues, callouts)
    # Next agent sees the updated vault from this agent's work
```

Key: `subagent_type: "general-purpose"` — NOT "Explore" — subagents need write access to vault.

Why sequential (not parallel):
- No write conflicts (each agent writes to vault before next starts)
- Each agent sees previous agents' work (vault updated incrementally)
- Dependency ordering: later agents reference already-documented modules
- Context isolation preserved (each agent has fresh context window)

### 1.3 Error Handling & Circuit Breaker

If subagent fails or times out:
1. Record failure: unit name, error type, error message
2. Continue with next unit — do not abort
3. Report failed unit in Phase 2

Circuit breaker (across iterations, tracked in state file):
- Track `failureHistory` per unit: count + lastError
- If a unit fails 2+ times across iterations -> mark as `permanentlyFailed`
- Permanently failed units are excluded from future iterations
- They still count toward completion assessment (see Phase 2.3)

---

## Phase 2 — Validation and Report

### 2.1 Validation

- **Broken wikilinks:** If `obsidianCli: true` in profile, run `obsidian unresolved vault={name} verbose`. Otherwise, grep vault for `[[wikilinks]]` and verify targets exist.
- **Index consistency:** All new docs linked from relevant `_Index.md` files
- **Bidirectional wikilinks:** If doc A links to doc B, verify B links back to A
- **Module map:** Check if architecture module map doc needs updating with new cross-module interactions (discover path from vault structure)

### 2.2 Update State File

Write/update `{vault}/_Meta/.full-research-state.yaml`:

```yaml
approvalPolicy: "skip-human"
promptFile: null
focusGuidance: "Focus on state machines"
lastRun: "2026-03-23T14:30:00"
iteration: 2
unitsCompleted: ["unit-1-name", "unit-2-name"]
unitsFailed: ["unit-3-name"]
failureHistory:
  "unit-3-name":
    count: 2
    lastError: "timeout"
permanentlyFailed: ["unit-3-name"]
```

### 2.3 Completion Detection & Summary

Re-scan the entire vault after all units complete. Re-classify all docs across all sections.

**If ALL vault docs (modules + architecture + reference + other sections) are deep AND no failed units:**

```
ALL_MODULES_DEEP

Full-project deep research complete — all {N} module docs are now deeply documented.

Statistics:
- Iterations: {iteration count from state file}
- Documents created: {X}
- Documents enriched: {Y}
- [!question] callouts: {Z} (require human review)
```

The `ALL_MODULES_DEEP` string serves as a machine-readable completion signal for automation (e.g., `/loop` or CI/CD pipelines).

**If some modules are still shallow/moderate OR units failed:**

```
Full-project deep research iteration {I} complete.

Results:
- Units researched: {M}/{N} ({K} skipped as deep, {F} failed)
- Documents created: {X}
- Documents enriched: {Y}
- Still shallow/moderate: {remaining count}
- [!question] callouts: {Z}

{For failed units: name + error summary}

Remaining work for next iteration:
{For each still-shallow: module name + current depth}

Run `/doc-full-research` again to continue with remaining modules.
```

---

## Tech-Agnosticity

Hard rules:
- Skill NEVER hardcodes framework-specific paths or patterns
- All project-specific knowledge comes from `project-profile.yaml`
- Subagent prompts are parametric — tech stack from profile drives search strategy
- Vault conventions (`conventions.md`) are tech-agnostic
- Module paths, directory names, and language are all from profile
- Dependency detection patterns come from `${CLAUDE_SKILL_DIR}/prompts/scope-decomposition.md` (per tech stack)
