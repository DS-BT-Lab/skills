# Documentation Conventions

## Table of Contents
- [Frontmatter](#frontmatter)
- [Tag Taxonomy](#tag-taxonomy)
- [Conditions and Code Expressions](#conditions-and-code-expressions)
- [Mermaid Diagrams](#mermaid-diagrams)
- [Naming Conventions](#naming-conventions)
- [Auto-Generatability](#auto-generatability)
- [Linking](#linking)

## Frontmatter

Every vault document has YAML frontmatter:

```yaml
---
type: module | adr | howto | tutorial | reference | explanation
status: draft | proposed | accepted | stale | deprecated | superseded
date: YYYY-MM-DD
generated: true              # true = AI generated, false/missing = human written
needs-review: true           # true = awaiting human review
tags: [...]                  # prefixed tags (see Tag Taxonomy)
aliases: []                  # optional, only if Omnisearch plugin detected
source: ...                  # only for migrated docs, path to original
superseded-by: ADR-XXXX      # only for ADR
supersedes: ADR-YYYY          # only for ADR
---
```

**Rules:**
- Keys always in English (Dataview/Bases compatibility). Content language follows user choice.
- `generated: true` is the key for refresh semantics — skill knows what it can safely overwrite.
- `needs-review: true` means awaiting human review. After user reviews, change to `false`.

## Tag Taxonomy

Prefixed hierarchy for clean filtering:

| Prefix | Purpose | Examples |
|--------|---------|----------|
| `type/` | Document type | `type/module`, `type/adr`, `type/howto` |
| `domain/` | Business domain | `domain/payment`, `domain/auth`, `domain/vehicle` |
| `layer/` | Technical layer | `layer/entity`, `layer/service`, `layer/controller` |

**Dataview/Bases queries MUST use frontmatter `type`** field (reliable). Tags with `type/` prefix serve visual navigation in Obsidian UI only.

## Conditions and Code Expressions

When documenting business logic conditions from code, every code expression **must** have a human-readable companion. Code alone is not sufficient — non-technical readers (PMs, business owners) cannot parse operators like `!=`, `&&`, `||`, `===`, or `null`.

### Format by Complexity

| Complexity | When | Readable form |
|------------|------|---------------|
| **Simple** (1–2 operators) | Single comparison or null check | Inline companion text next to the code |
| **Medium** (3–5 operators) | Combined conditions (AND/OR) | Bulleted list of individual conditions |
| **Complex** (5+ or multi-variable) | Decision logic with multiple inputs/outputs | Decision table |

### Simple — inline companion

Place the human-readable meaning first, code in parentheses:

```markdown
The order is confirmed when it has an assigned number (`number !== null`).
```

Not:
```markdown
`number !== null` means the order is confirmed.
```

### Medium — bulleted list

Break compound conditions into individual items, each with its meaning and code reference:

```markdown
The button is visible when all conditions are met:
- Start date is set (`dateFrom`)
- End date is set (`dateTo`)
- Remaining quantity is greater than zero (`remaining > 0`)
```

### Complex — decision table

Use a table with a human-readable column first, code column second:

```markdown
| Condition | Code | Required |
|-----------|------|----------|
| Not yet invoiced | `invoiced != true` | yes |
| Not manually invoiced | `manually_invoiced != true` | yes |
| Order is complete | `completed = true` | yes |
```

### Rules

- **Always lead with meaning, follow with code** — the readable form comes first, code serves as a precise reference for developers.
- **Preserve code expressions** — do not remove or replace them. They are the source of truth for developers.
- **Use the project's `language`** from `project-profile.yaml` for the readable text. Code stays in its original language.
- **In Mermaid diagrams**, keep node labels readable. If a node contains code conditions, add a companion text node or use the label for the human-readable form and an annotation for the code.

## Mermaid Diagrams

### Limits

| Rule | Value | Reason |
|------|-------|--------|
| Max nodes per diagram | **50** | Cognitive readability limit |
| Max parallel branches | **8** | Beyond 8 becomes unreadable |
| ELK renderer | For **20+ nodes** | Better automatic layout |

**ELK directive** (add at start of Mermaid block):
```
%%{init: {"flowchart": {"defaultRenderer": "elk"}}}%%
```

### Diagram Type Selection

| Type | Use for | Example |
|------|---------|---------|
| Flowchart | Process flow, decision trees | "How a request flows through the system" |
| Sequence | Component interactions over time | "What happens when a webhook arrives" |
| Class | Entity relationships, inheritance | "Vehicle entity structure" |
| ER | Database schema with cardinality | "Table relationships" |
| State | Entity lifecycle | "Reservation states" |

### Split Strategy for Large Diagrams

1. **Per module** — overview diagram (boxes = modules) + detail diagram per module
2. **Per use-case** — one sequence diagram per use-case, not one giant diagram
3. **Per layer** — separate diagram for entities, services, controllers
4. **Cross-reference** — overview names sub-systems, detail diagrams in separate files with `[[wikilinks]]`

## Naming Conventions

**File names:** `PascalCase_With_Underscores.md` (e.g., `System_Overview.md`).
**Module files:** Use codebase name (e.g., `Payments.md`, `AuthService.md`, `UserStore.md`).
**ADR files:** `ADR-0001_Use_Explicit_Service_Registration.md` (4-digit zero-padded).
**Index files:** `_Index.md` in each content directory.
**Dashboard:** `00_Home.md` at vault root.

## Auto-Generatability

| Confidence | Content Type | `needs-review` | Examples |
|-----------|-------------|-----------------|----------|
| **HIGH** | Structural reference | `false` | Entity_Map.md, Tech_Stack.md, dependency diagrams |
| **HIGH** | Indexes and overviews | `false` | Module_Map.md, _Index.md, 00_Home.md |
| **MEDIUM** | Module descriptions | `true` | Purpose and dependencies AI estimates, detail needs human |
| **MEDIUM** | ADR drafts | `true` | From code + CLAUDE.md, rationale needs review |
| **LOW** | Explanations | `true` | "Why we chose X" — requires human context |
| **NEVER** | Tutorials | — | Pedagogical design — human only |

Generate only HIGH and MEDIUM confidence content. Explanations only if strong context exists.

## Linking

**Obsidian mode:** Use `[[wikilinks]]` for internal vault references. Dense linking improves Graph view.

**Plain Markdown mode:** Use `[display text](relative/path.md)` for internal references. Same linking density, just different syntax.

**External references:** Use standard `[text](url)` or relative paths to files outside vault.