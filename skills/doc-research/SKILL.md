---
name: doc-research
description: >
  Use when you need deeper understanding of a specific codebase area — module
  deep-dive, cross-cutting analysis, or enriching existing vault documents.
  Requires an existing vault from /init-docs.
argument-hint: "<topic or question to research>"
user-invocable: true
disable-model-invocation: true
---

# /doc-research — Deep-Dive Documentation Research

Deep-dive research into a codebase topic, producing new vault documents or enriching existing ones.

**Usage:** `/doc-research <topic or question>`
- `/doc-research Deep-dive do Delivery modulu — stavovy automat, PDF generovani`
- `/doc-research Prehled vsech admin modulu z business pohledu`
- `/doc-research Authentication and authorization flow`
- `/doc-research Analyza API endpointu pro e-shop integraci`
- `/doc-research State management architecture`
- `/doc-research Jak funguje fakturacni system — typy faktur, workflow, vazby`

User context from `$ARGUMENTS` drives the scope, intent, and depth of research.

**Prerequisite:** Vault must exist with `_Meta/project-profile.yaml`. If not found, stop with error: "Run `/init-docs` first to generate the vault."

## Supporting Files

Read these on-demand (progressive disclosure):

| File | When to read | Purpose |
|------|-------------|---------|
| `${CLAUDE_SKILL_DIR}/../init-docs/references/conventions.md` | Phase 0, Phase 5 | Frontmatter, tags, Mermaid, naming conventions |
| `${CLAUDE_SKILL_DIR}/../init-docs/references/diataxis-guide.md` | Phase 1 (intent mapping) | Diataxis framework reference |
| `${CLAUDE_SKILL_DIR}/../init-docs/templates/module.md` | Phase 5 (new module docs) | Module document template |
| `${CLAUDE_SKILL_DIR}/../init-docs/templates/adr.md` | Phase 5 (if ADR needed) | ADR template |
| `${CLAUDE_SKILL_DIR}/../init-docs/templates/howto.md` | Phase 5 (if howto needed) | Guide template |
| `${CLAUDE_SKILL_DIR}/prompts/research-analysis.md` | Phase 3 (broad topics) | Subagent prompt for Explore agents |

## Progress Tracking

Use TaskCreate/TaskUpdate for real-time progress:
- "Phase 0: Loading vault context..."
- "Phase 1: Analyzing scope..."
- "Phase 2: Auditing existing vault docs..."
- "Phase 3: Researching codebase (subagent 1/3)..."
- "Phase 5: Generating documents — 3/5 done..."

Between phases, briefly inform the user of the previous phase's result (1-2 sentences).

## Quality Standards

### Anti-Hallucination Principle

**This skill NEVER speculates.** Every statement must be backed by a specific file from the codebase.

| Situation | Action |
|-----------|--------|
| Fact determinable from code | State directly, cite source file |
| Cannot determine from code | Obsidian callout `[!question]` explaining what's missing and where you looked |
| Business reason, motivation, historical context | NEVER invent — always `[!question]` callout |

These rules propagate to subagent prompts — subagents return only facts + explicit "unknown" for uncertainties.

Example callout:

```markdown
> [!question] K doplneni
> Z kodu neni jasne, proc je `DeliveryFingerprintingService` implementovan
> vlastnim resenim misto standardniho hashovaciho mechanismu.
> Nalezeno v: `src/Services/Delivery/DeliveryFingerprintingService.php`
```

### needs-review by Content Reliability

| Reliability | Example | `needs-review` |
|-------------|---------|-----------------|
| **High** — directly from code | "DeliveryAdmin has 47 form fields, 3 batch actions" | `false` |
| **High** — structure/relations | "Delivery depends on SalesOrder via ManyToOne" | `false` |
| **Medium** — derived from code | "Module manages delivery logistics" | `true` |
| **Low** — motivation/context | "Why custom ValidDelivery instead of standard constraints" | `true` |

Document-level flag = lowest reliability in the document. If one section is medium, the entire doc gets `needs-review: true`.

### Audience Awareness

Purpose/Účel sections must be readable by non-technical stakeholders (PO, PM, business owners). No class names, no framework terms, no code references — business terminology only. Technical depth belongs in subsequent sections (How it works, Key components).

### Context7 MCP

If context7 MCP is available, use it to verify framework APIs and conventions:
- `resolve-library-id` to identify the library
- `query-docs` to fetch current documentation for accurate API descriptions

Increases reliability, reduces hallucinations. Subagent prompts include instruction to verify framework concepts via context7 when available.

## Document Ownership Rules

### Permissions

| Action | doc-research |
|--------|-------------|
| Create new document | Yes (companion docs) |
| Overwrite `generated: true` + `needs-review: true` | Yes |
| Modify `generated: true` + `needs-review: false` | Yes (with evidence) |
| Modify `generated: false` | **Yes, with per-document approval** |
| Update Mermaid diagrams | Yes |

### Rules for `generated: false` Documents

If a factual error is found in a human-authored document, it CAN be fixed, but:

1. Phase 4 plan explicitly marks it as a human document
2. Requires **per-document approval** (not bulk)
3. Without explicit consent, do not modify

Example in plan:

```
!! Lidsky dokument — vyzaduje schvaleni:
3. 50_Reference/Fakturace_Helianthus.md (generated: false)
   - OPRAVIT: ciselna rada 2026 chybi, pridana v kodu (InvoiceNumberType.php:45)
   Chcete provest opravu? [ano/ne]
```

### How doc-research Changes Flags

| Change Type | Flag After Change | init-docs refresh behavior |
|------------|-------------------|---------------------------|
| Error fix (wrong path, incorrect relation) | Keep existing | Overwrites (fixes too) |
| Structural enrichment (high reliability) | `needs-review: false` | Compares, reports, doesn't overwrite |
| Behavior/motivation description (medium/low) | `needs-review: true` | Overwrites (OK — content wasn't validated) |
| New companion doc | `generated: true`, per reliability | Ignores (doesn't know about it) |

---

## Autonomous Mode

If `$ARGUMENTS` contains `--autonomous`, the skill runs without user interaction. All research quality standards, anti-hallucination rules, and ownership rules still apply.

**Activation:** `--autonomous` flag anywhere in `$ARGUMENTS`. Parse and remove from the research topic string.

**Supported flags:**
- `--autonomous` — enable autonomous mode
- `--policy skip-human` (default when autonomous) — skip `generated: false` documents entirely
- `--policy auto-all` — approve everything including `generated: false`
- `--policy preserve-human` — enrich `generated: false` but preserve existing structure

**Phase behavior changes:**

| Phase | Normal (interactive) | Autonomous |
|-------|---------------------|------------|
| Phase 1 (Scope) | Ask if intent is ambiguous | Infer from topic + scope context. Never ask. |
| Phase 4 (Plan) | Present plan, wait for approval | Auto-approve per policy. Skip presentation. |
| Phase 6 (Report) | Report to user, do NOT commit | Report summary to caller. Do NOT commit. |

**Argument parsing:** Strip `--autonomous` and `--policy <value>` from `$ARGUMENTS` before using remainder as research topic.

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

## Phase 1 — Scope Analysis

Analyze `$ARGUMENTS` and classify the research topic.

**Breadth:**
- **Broad** — cross-cutting topic spanning multiple modules (e.g., "admin modules overview", "API endpoints")
- **Focused** — single module or narrow topic (e.g., "delivery state machine")

**Intent (maps to Diataxis):**
- **What/Which** — reference document (facts, structure, overviews)
- **How** — howto document (steps, procedures)
- **Why** — explanation document (reasoning, context)
- **Detail** — enriched module doc (extending existing)

Read `${CLAUDE_SKILL_DIR}/../init-docs/references/diataxis-guide.md` if intent mapping is ambiguous.

**Relevant tech stacks:** From `project-profile.yaml` — if the topic spans multiple stacks (e.g., PHP backend + Vue frontend), identify both.

**Relevant modules:** Which modules from the profile relate to the topic.

No subagent needed for scope analysis — this is classification from the argument + profile.

Output (structured, internal):

```yaml
scope:
  breadth: broad | focused
  intent: reference | howto | explanation | enrichment
  relevantStacks: ["PHP/Symfony", "JavaScript/Vue.js"]
  relevantModules: ["Delivery", "Invoicing"]
  keywords: ["state machine", "delivery states"]
```

If the intent is ambiguous, ask the user before proceeding.

---

## Phase 2 — Vault Audit

Search the vault for existing coverage of the topic:

- Grep for topic keywords in document content
- Frontmatter tags (`domain/`, `layer/`) matching the topic
- Wikilinks to relevant modules (`[[Module_Name]]`)
- `_Index.md` in relevant directories

For each relevant document found, read its content and assess:

| Rating | Meaning |
|--------|---------|
| **Sufficient** | Topic covered with adequate depth |
| **Shallow** | Topic mentioned but lacks depth |
| **Inaccurate** | Contains errors or outdated information |
| **Missing** | Topic not in vault at all |

---

## Phase 3 — Research

Deep codebase analysis. Strategy depends on topic breadth:

| Breadth | Strategy |
|---------|----------|
| **Focused** (one module, one topic) | Skill itself — Glob + Grep + Read targeted files |
| **Broad** (cross-cutting, multiple modules) | Dispatch 1+ Explore subagents in parallel |

### Focused Research

Use Glob, Grep, and Read directly. Start from module paths in `project-profile.yaml`, trace dependencies outward. Read method implementations, not just signatures. Check edge cases and error paths.

### Broad Research (Subagent Dispatch)

1. **Read** `${CLAUDE_SKILL_DIR}/prompts/research-analysis.md`
2. **Construct** the full agent prompt:
   - Prompt file content (role + task + strategy + output format)
   - `---` separator
   - `## Project Profile` header followed by `project-profile.yaml` content
   - `## Research Scope` header followed by Phase 1 scope analysis
   - `## User Context` header followed by `$ARGUMENTS`
3. **Dispatch** via Agent tool with `subagent_type: "Explore"`

**Splitting strategy for broad topics:**
- Guided by `project-profile.yaml` modules[]
- Group modules by relevance to topic
- Dispatch one subagent per group
- For cross-cutting (non-module) topics: split by aspect (e.g., "entities" vs "services" vs "admin")

**Multi-stack topics:** If the topic spans multiple tech stacks (backend + frontend):
- Dispatch one subagent per stack, OR
- One subagent with explicit instruction to explore both sides

**Error handling:** If any subagent fails or times out, continue with available data. Note missing analyses in Phase 6 report.

---

## Phase 4 — Plan (Interactive)

Present research results and proposed changes to the user:

```
Research results for topic "admin modules":
- Found 47 admin registrations in 6 domain groups
- Vault has: 12 module docs (shallow admin interface mentions)
- Missing: unified business perspective overview

Proposed changes:
1. CREATE: 50_Reference/Admin_Moduly.md (type: reference)
   - Complete overview of all admin modules, table, Mermaid workflow
2. UPDATE: 20_Moduly/Dodavky.md (generated: true, needs-review: true)
   - Add: batch operations in admin, custom controller actions
!! HUMAN DOC — requires approval:
3. 50_Reference/Fakturace_Helianthus.md (generated: false)
   - Fix: missing 2026 numbering series
   Approve this change? [yes/no]

Continue? [yes / adjust scope]
```

**Key rules:**
- Target vault directory is proposed HERE (after research), not in Phase 1
- The Diataxis-appropriate directory is chosen based on research findings + intent
- For `generated: false` documents, require per-document approval
- Wait for user confirmation before proceeding to Phase 5

---

## Phase 5 — Generate

Read `${CLAUDE_SKILL_DIR}/../init-docs/references/conventions.md` before generating.

For each approved change:

- **New documents:** Use template from `${CLAUDE_SKILL_DIR}/../init-docs/templates/` + research data. Set frontmatter per conventions.
- **Modifications to existing:** Make precise, evidence-backed changes.
- **Wikilinks:** Bidirectional — add link in the new/modified doc AND in the target document.
- **Mermaid diagrams:** Per conventions (max 50 nodes, ELK for 20+ nodes, split by topic). Use `language` from profile for node labels.
- **Index updates:** Update relevant `_Index.md` files to include new documents.
- **Obsidian callouts:** `[!question]` for every uncertainty (see Anti-Hallucination Principle).

**Flag rules:** See "How doc-research Changes Flags" above.

### Pre-completion Self-check

Before proceeding to Phase 6, verify for each generated/modified document:
- All mentioned file paths exist (Glob check)
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
   - What was created, what was modified
   - All `[!question]` callouts (areas requiring human input)
   - Suggested follow-up topics for further research
3. **Do NOT commit** — the user commits when ready.

---

## Tech-Agnosticity

Hard rules:

- Skill NEVER hardcodes framework-specific paths or patterns
- All project-specific knowledge comes from `project-profile.yaml` (tech stack, `modules[].path`, language)
- Subagent prompts are parametric — they receive tech stack from profile and choose search strategy themselves
- Vault conventions (`conventions.md`) are tech-agnostic
