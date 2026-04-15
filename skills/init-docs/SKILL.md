---
name: init-docs
description: >
  Use when starting documentation for a new codebase, or refreshing an existing
  vault after significant project changes. Covers any tech stack — detects
  modules, dependencies, and architecture automatically.
argument-hint: "[optional context or instructions]"
user-invocable: true
disable-model-invocation: true
---

# /init-docs — Second Brain Documentation Generator

Generate structured documentation for any codebase into an Obsidian vault or plain Markdown.

**Usage:** `/init-docs [optional context]`
- `/init-docs` — analyze current project, use defaults
- `/init-docs Focus on payment and auth modules` — scope to specific areas
- `/init-docs Vault is in ./documentation/brain/` — specify vault location

User context from `$ARGUMENTS` is available to all phases as supplementary instructions.

## Supporting Files

Read these from this skill's directory on-demand (progressive disclosure):

| File | When to read | Purpose |
|------|-------------|---------|
| `${CLAUDE_SKILL_DIR}/references/diataxis-guide.md` | Phase 2 (classification), Phase 4 | Diataxis framework reference |
| `${CLAUDE_SKILL_DIR}/references/conventions.md` | Phase 4 (generation) | Frontmatter, tags, Mermaid, naming conventions |
| `${CLAUDE_SKILL_DIR}/templates/module.md` | Phase 1 (scaffold), Phase 4 (generation) | Module document template |
| `${CLAUDE_SKILL_DIR}/templates/adr.md` | Phase 1 (scaffold), Phase 4 (generation) | ADR template (MADR format) |
| `${CLAUDE_SKILL_DIR}/templates/howto.md` | Phase 1 (scaffold) | Guide template |
| `${CLAUDE_SKILL_DIR}/prompts/discovery-modules.md` | Phase 3 (discovery) | Subagent prompt: module analysis |
| `${CLAUDE_SKILL_DIR}/prompts/discovery-deps.md` | Phase 3 (discovery) | Subagent prompt: dependency mapping |
| `${CLAUDE_SKILL_DIR}/prompts/discovery-infra.md` | Phase 3 (discovery) | Subagent prompt: infrastructure |
| `${CLAUDE_SKILL_DIR}/prompts/discovery-adr.md` | Phase 3 (discovery) | Subagent prompt: ADR candidates |

## Progress Tracking

Use TaskCreate/TaskUpdate to show real-time progress:
- "Phase 0: Recognizing project..."
- "Phase 3: Discovery — analyzing modules (subagent 1/4)..."
- "Phase 4: Generating documents — 12/31 modules done..."

Between phases, briefly inform the user of the previous phase's result (1-2 sentences).

## First Run vs Refresh

Check if vault already contains `00_Home.md`:
- **First run** (`00_Home.md` missing): Execute all phases 0–5.
- **Refresh** (`00_Home.md` exists): Re-run Phase 0, **skip Phases 1–2**, refresh Phases 3–5. See "Refresh Semantics" section below.

---

## Phase 0 — Project Recognition

**Goal:** Build a project profile that drives all subsequent phases.

### 0.1 Vault Detection

Search for vault in order:
1. Path from `$ARGUMENTS` (if user specified one)
2. `docs/Vault/` (default location)
3. `docs/Vault-*/` (glob for project-named vaults)

If directory exists and contains `.obsidian/` → Obsidian vault mode (`obsidianMode: true`).
If not found, ask:

> "I didn't find an Obsidian vault. Options:
> a) Create Obsidian vault (with `.obsidian/` config, wikilinks, Dataview queries)
> b) Generate plain Markdown (same structure, relative links instead of wikilinks, static lists instead of Dataview)
> Where to place documentation? [default: docs/Vault/]"

**Plain Markdown mode** means: `[[wikilinks]]` → `[name](relative/path.md)`, Dataview blocks → static lists, no `.obsidian/` config, no Bases files.

### 0.2 Tech Stack Detection

Recursively search for manifest files. **Respect .gitignore.** Exclude: `node_modules/`, `vendor/`, `.git/`, `build/`, `dist/`, `cache/`.

| Manifest | Stack |
|----------|-------|
| `composer.json` | PHP + framework from `require` section |
| `package.json` | Node.js + framework from `dependencies` |
| `requirements.txt` / `pyproject.toml` | Python + framework |
| `go.mod` | Go |
| `Cargo.toml` | Rust |

**Version resolution:** Read each library's version individually from its manifest entry (`require`, `dependencies`). Never assign one version to multiple libraries. If a tech stack row groups libraries (e.g., "Vue.js + Vuex"), resolve and report each version separately.

**Monorepo:** If manifests found at multiple directory levels → monorepo. If **15+ packages** detected, ask:
> "Detected monorepo with [N] packages. Options:
> a) Document all (may take longer)
> b) Select specific packages — which? [list]
> c) Root architecture + package overview only (no deep-dive)"

### 0.3 Module Structure Detection

Three levels, applied in order:

1. **Framework-specific:** Symfony bundles (`*Bundle/`), npm workspaces, Django apps (`apps.py`), Go packages, monorepo packages
2. **Generic heuristic:** 3+ sibling directories under a common parent with shared internal structure (e.g., each has `Domain/` + `Application/`, or each has `index.ts` + `components/`)
3. **Flat:** No module boundaries detected — project has flat organization

### 0.4 Architecture Style Detection

Detect from directory structure, namespace conventions, and configurations. **Multiple styles can coexist** (e.g., hexagonal backend + component-based frontend):

| Style | Indicators |
|-------|-----------|
| MVC | controllers/, models/, views/ |
| Hexagonal | Domain/, Application/, Infrastructure/ (top-level or nested per component) |
| CQRS | Command/, Query/, CommandHandler/, QueryHandler/ |
| DDD | Isolated bounded contexts with own entities, services, events |
| Microservices | Multiple independent services with own manifests |
| API-only | No views/templates, controllers + serializers/transformers only |
| Server-side rendered | Twig/Blade/EJS templates with minimal JS logic |
| Component-based FE | components/, hooks/composables/, stores/, pages/ |
| Mixed/Islands | Server-side templates + embedded client components (e.g., PHP/Twig + Vue) |
| SPA | Single-page app with client-side routing |
| SSR/SSG | Next.js/Nuxt app router, server/client component separation |

Architecture style affects module organization in Phase 4 and which architecture-specific documents to generate.

### 0.5 Existing Documentation

Find markdown files in `docs/`, `documentation/`, root README files, CLAUDE.md, AGENTS.md. For each, record path and estimate probable Diataxis type (reference `${CLAUDE_SKILL_DIR}/references/diataxis-guide.md` for classification rules).

### 0.6 Obsidian Plugins (if obsidianMode)

Read `.obsidian/community-plugins.json`, `.obsidian/core-plugins.json`, and list `.obsidian/plugins/` directory.

Adaptive behavior based on detected plugins:

| Plugin | If found | If missing |
|--------|----------|-----------|
| Dataview | Generate `dataview` query blocks in indexes and dashboard | Generate static lists with wikilinks |
| Bases (core) | Generate `.base` files for modules and ADR views | Skip |
| Templates (core) | Configure `_Templates/` path in `.obsidian/templates.json` | Create templates without Obsidian integration |
| Excalidraw | Mention interactive diagram option in review report | Skip |
| Omnisearch | Add `aliases` field to frontmatter | Skip aliases |
| Bookmarks | Suggest key bookmarks in review report | Skip |
| Graph | Ensure dense `[[wikilinks]]` network for graph visualization | Wikilinks generated regardless |

### 0.7 Additional Detections

- **Obsidian CLI:** Run `which obsidian` — if available, can use for `property:set`, `search`, `backlinks` operations. Always optional — fallback to direct file I/O.
- **Project type:** full-stack | API-only | library | CLI tool (from routing, controllers, views presence)
- **Cross-repo refs:** From CLAUDE.md/README mentions, API client configs (base URLs), git submodules, OpenAPI/Swagger specs referencing external services
- **Project context:** Summarize from CLAUDE.md, README, `git log --oneline -20` — what's being actively worked on

### 0.8 Documentation Language

Ask user:
> "Documentation language? (affects content, folder names, template sections)
> Detected: [language from existing docs / CLAUDE.md conventions]
> a) Czech  b) English  c) Other — specify
> [default: detected language]"

### 0.8b File Naming Convention

Ask user:
> "File naming convention? (how to name generated .md files)
> a) System/UI names — match domain terminology (e.g., Bestellungen.md, Utilisateurs.md)
> b) Code names — match codebase identifiers (e.g., OrderModule.md, UserService.md)
> c) Mixed — system names for domain modules, code names for technical docs
> [default: a]"

Save choice to project-profile.yaml as `fileNaming: system | code | mixed`.
In Phase 4, apply this convention when naming generated files:
- `system`: translate module names to the chosen language, use domain terminology
- `code`: use entity/class/package names as-is from the codebase
- `mixed`: domain modules use system names, architecture/reference docs use code names

### 0.9 Project Profile Output

Save to `{vault}/_Meta/project-profile.yaml`:

```yaml
# generated: true
techStack:
  - { language: "TypeScript", framework: "Next.js 14", version: "5.3", packageManager: "pnpm" }
  - { language: "Python", framework: "FastAPI", version: "3.12", packageManager: "uv" }
architectureStyle: "component-based"
projectType: "full-stack"
modules:
  - { name: "Dashboard", path: "src/features/dashboard/", type: "feature" }
  - { name: "Auth", path: "src/features/auth/", type: "feature" }
existingDocs:
  - { path: "docs/architecture.md", probableType: "explanation" }
crossRepoRefs:
  - { name: "API Gateway", url: "https://github.com/org/api-gateway" }
obsidianMode: true
obsidianCli: false
vaultPlugins:
  dataview: true
  templates: true
  bases: false
  excalidraw: false
  omnisearch: false
  bookmarks: false
  graph: true
language: "en"
fileNaming: "code"
projectContext: "SaaS platform for team collaboration..."
```

---

## Phase 1 — Scaffold

**Skip on refresh (vault has `00_Home.md`).**

### 1.1 Directory Structure

Create vault directories. Names adapt to chosen language:

| English (default) | Czech variant |
|-------------------|--------------|
| `10_Architecture/` | `10_Architektura/` |
| `20_Modules/` | `20_Moduly/` |
| `30_Decisions/` | `30_Rozhodnuti/` |
| `40_Guides/` | `40_Navody/` |
| `50_Reference/` | `50_Reference/` |
| `_Templates/` | `_Sablony/` |
| `_Meta/` | `_Meta/` |

### 1.2 Index Files

Create `_Index.md` in each content directory (10–50) with frontmatter: `type: reference`, `generated: true`, `needs-review: false`.
- If Dataview plugin → dynamic query listing documents in that folder filtered by type
- Else → static placeholder list (populated in Phase 4)

### 1.3 Vault Templates

Read templates from `${CLAUDE_SKILL_DIR}/templates/` directory. Create translated copies in vault's `_Templates/` (or `_Sablony/` for Czech):
- Templates use `<<placeholder>>` syntax (distinct from Obsidian's `{{}}` variables)
- Replace `<<date>>` with Obsidian template variable `{{date}}`
- Translate `<<section_*>>` headers to chosen language as fixed text
- Convert other `<<placeholders>>` to instructional text in chosen language
- Keep frontmatter keys in English

### 1.4 Obsidian Configuration

**Only if obsidianMode.** Modify **only** `.obsidian/templates.json` — merge the `folder` key, preserve other settings:
```json
{ "folder": "_Templates", "dateFormat": "YYYY-MM-DD", "timeFormat": "HH:mm" }
```
**Do NOT modify any other `.obsidian/` files.** Other recommendations go in the Phase 5 review report.

### 1.5 Vault .gitignore

Create `{vault}/.gitignore` if it does not already exist:

```gitignore
# Obsidian - volatile UI state
.obsidian/workspace.json
.obsidian/workspace-mobile.json
.obsidian/graph.json
.trash/

# Obsidian - plugin code (downloadable, large)
.obsidian/plugins/*/main.js
.obsidian/plugins/*/styles.css
.obsidian/plugins/*/manifest.json

# Commit: community-plugins.json, core-plugins.json (plugin lists)
# Commit: plugins/*/data.json (plugin settings)
# Commit: app.json, appearance.json, templates.json (core config)
```

### 1.6 Dashboard

Create `00_Home.md` (`generated: true`, `needs-review: false`) — dashboard with quick links and overviews. If Dataview → dynamic queries. Else → static links (populated in Phase 4).

### 1.7 Bases Files

**Only if Bases plugin detected.** Create:
- `Modules.base` — table view of `type: module` documents in `20_Modules/`
- `Decisions.base` — table view of `type: adr` documents in `30_Decisions/`

---

## Phase 2 — Migration of Existing Docs

**Skip on refresh or if no existing docs found in Phase 0.**

Read `${CLAUDE_SKILL_DIR}/references/diataxis-guide.md` for classification rules.

For each existing documentation file found in Phase 0:
1. Read its content. Classify its Diataxis type (tutorial, howto, reference, explanation).
2. Map to target vault directory based on type.
3. Transform: add frontmatter (`generated: false`, `source: original/path.md`), convert links to `[[wikilinks]]` (or relative paths if plain Markdown mode).
4. Write transformed copy to vault. **Do NOT modify the original file.**

**Rules:**
- Non-markdown files (code examples, images, scripts): do NOT migrate. Link from vault docs using relative paths.
- Auto-generated docs (OpenAPI specs, PHPDoc output, TypeDoc): do NOT duplicate. Link from vault docs.
- When extending an existing `_Index.md`, add content **below** the scaffold content, not replacing it.

**Plans and specs:** If docs that look like live plans are found (in `docs/plans/`, `docs/specs/`, etc.), ask:
> "Found these plan/spec documents: [list]. Which are:
> a) Live (I'll link from the vault, not migrate)
> b) Historical (I'll migrate to 30_Decisions/ or 10_Architecture/)"

---

## Phase 3 — Discovery

Dispatch **4 parallel subagents** for deep codebase analysis.

For each prompt file in `${CLAUDE_SKILL_DIR}/prompts/`:
1. **Read** the prompt file content from this skill's directory
2. **Construct** the full agent prompt in this exact order:
   - Prompt file content (role + task + strategy + output format)
   - `---` separator
   - `## Project Profile` header followed by the project-profile.yaml content
   - `## User Context` header followed by `$ARGUMENTS` (if provided)
3. **Dispatch** via Agent tool with `subagent_type: "Explore"`

**All 4 agents launch in parallel:**

| # | Prompt file | Agent description |
|---|------------|-------------------|
| 1 | `${CLAUDE_SKILL_DIR}/prompts/discovery-modules.md` | "Discover project modules" |
| 2 | `${CLAUDE_SKILL_DIR}/prompts/discovery-deps.md` | "Map module dependencies" |
| 3 | `${CLAUDE_SKILL_DIR}/prompts/discovery-infra.md` | "Analyze infrastructure" |
| 4 | `${CLAUDE_SKILL_DIR}/prompts/discovery-adr.md` | "Find ADR candidates" |

**Error handling:** If any subagent fails or times out, continue with data from successful agents. Note missing analysis in Phase 5 review report.

If `$ARGUMENTS` mentions additional tools (Chrome MCP, etc.), include that info in the subagent prompts.

---

## Phase 4 — Generation

**Input:** Discovery results from Phase 3 + project profile from Phase 0.
**Read** `${CLAUDE_SKILL_DIR}/references/conventions.md` for frontmatter, tags, Mermaid, and naming conventions.

**Empty results:** If a discovery area returns empty data (no modules, no dependencies, no ADR candidates), generate its `_Index.md` with a note that nothing was detected, and mention it in the Phase 5 review report as an area for human input.

### 4.1 Architecture Documents (10_Architecture/)

**Always generate:**
- **System_Overview.md** — what the project is, who it serves, tech stack, high-level Mermaid component diagram. `needs-review: false`
- **Module_Map.md** — all modules with Mermaid dependency diagram (from discovery-deps data), `[[wikilinks]]` to each module doc. `needs-review: false`

**If cross-repo refs found:**
- **External_Dependencies.md** — map of external systems, API contracts, relationships. `needs-review: true`

**Architecture-specific documents** (`needs-review: true`):

| Detected Style | Generate |
|---------------|----------|
| Hexagonal | `Ports_And_Adapters.md`, `Layer_Dependencies.md` |
| CQRS | `Command_Query_Flow.md`, `Read_Write_Models.md` |
| DDD | `Context_Map.md`, `Ubiquitous_Language.md` |
| API-only | `API_Surface.md` (per portal if multiple detected), `API_Contracts.md` |
| Microservices | `Service_Map.md`, `Inter_Service_Contracts.md` |
| Component-based FE | `Component_Tree.md`, `State_Management.md` |
| Mixed (BE+FE) | Backend docs + frontend docs + `BE_FE_Integration.md` |
| SSR/SSG framework | `Routing.md`, `Server_Client_Boundary.md` |

Also generate a document for each cross-cutting pattern that discovery-modules marked as `needsArchDoc: true`.

### 4.2 Module Documents (20_Modules/)

Read `${CLAUDE_SKILL_DIR}/templates/module.md`. For each discovered module, generate a document:
- Translate template section headers to chosen language
- Fill from discovery data: purpose, key components, dependencies, status
- Add Mermaid diagram if module complexity is medium or high (3+ entities/components)
- Set frontmatter per `${CLAUDE_SKILL_DIR}/references/conventions.md` (type, status, date, generated, needs-review, tags)

**Organization depends on architecture style:**

| Style | Organization |
|-------|-------------|
| MVC / Bundle-based | Flat — one file per module |
| Hexagonal / DDD | Per bounded context, with layer tags in frontmatter |
| CQRS (top-level) | Per context with command/query separation files |
| CQRS (inline) | CQRS as sections within the module document |
| Microservices | Per service |
| API-only | Per resource/endpoint group |
| Component-based FE | Per feature/page + shared component groups |
| Mixed (BE+FE) | `Backend/` and `Frontend/` sub-directories |

If no modules were identified (flat project), `20_Modules/` contains only `_Index.md` with a note that the project has flat structure.

### 4.3 Decision Records (30_Decisions/)

Read `${CLAUDE_SKILL_DIR}/templates/adr.md`. For each ADR candidate from discovery-adr:
- Generate ADR document in MADR minimal format
- Number: 4-digit zero-padded (`ADR-0001`, `ADR-0002`, ...)
- File name: `ADR-0001_Decision_Title.md`
- Set `status: proposed`, `needs-review: true`
- **Anti-hallucination:** If business reason unknown, write: "Detected pattern X. Business reason unknown — please supplement with context."

### 4.4 Reference Documents (50_Reference/)

**Always generate:**
- **Tech_Stack.md** — versions, configurations, where to find things, key commands. `needs-review: false`

**Generate if relevant** (from discovery-infra data): service registry, entity/model map, API endpoints reference, environment setup guide.

### 4.5 Guides (40_Guides/)

**Do NOT generate new guides.** Only migrated docs from Phase 2 go here. `_Index.md` links to any migrated guides. If no guides were migrated, `_Index.md` has a note that guides are added manually.

### 4.6 Dashboard + Index Updates

Update `00_Home.md` with:
- Quick links to all generated architecture and reference documents
- Vault statistics (total modules, ADRs, guides, diagrams)
- If Dataview → dynamic queries; else → static links

Update all `_Index.md` files to link to generated documents in their directories.

### 4.7 Inline Validation (if obsidianCli)

If `obsidianCli: true` in project profile, run validation after all documents are generated:

1. `obsidian unresolved vault=<name> verbose` — list broken wikilinks
2. `obsidian orphans vault=<name> total` — count orphaned files
3. `obsidian files vault=<name> total` — total file count

If unresolved links found (excluding template placeholders like `<<placeholder>>`):
- Attempt auto-fix: create missing target documents or correct wikilink spelling
- Re-run validation (max 2 iterations)

If orphaned files exceed 20% of total files, report as warning in Phase 5.

Include validation results (resolved count, remaining issues, orphan count) in Phase 5 review report under a **Vault Health** section.

---

## Phase 5 — Review Report

Present a structured report to the user in conversation:

1. **Statistics** — documents created, migrated, modules documented, diagrams generated
2. **ADR candidates for approval** — list each with evidence, ask "Create ADR?" per candidate
3. **Uncertainties** — modules with unclear purpose, ambiguous dependencies, unresolved questions
4. **Next steps** — modules deserving deep-dive (high complexity, active development)
5. **Plugin recommendations** — if key plugins missing: Dataview (strongly recommended), Omnisearch, Excalidraw (optional)
6. **Bookmark suggestions** — if Bookmarks plugin detected, suggest key vault documents to bookmark
7. **Stale documents** (refresh only) — documents where codebase has diverged from content, suggest `status: stale`
8. **Bases views** — if Bases plugin detected, suggest additional useful views
9. **Failed analyses** — if any Phase 3 subagent failed, report what analysis is missing
10. **Unmapped areas** — if discovery-modules reported `unmappedAreas`, mention them as areas that may need manual documentation
11. **Layer violations** — if discovery-deps reported `layerViolations`, include them for hexagonal/DDD projects or note them for other architectures
12. **Vault health** (if obsidianCli) — unresolved wikilinks remaining after auto-fix, orphaned files count, total file count

**After user feedback:**
- Create/update ADRs based on user's answers
- Supplement uncertainties with user-provided context
- **Do NOT commit** — the user commits when ready

---

## Refresh Semantics

When vault already contains `00_Home.md` (repeat run):

| Document state | Action |
|---------------|--------|
| `generated: true` + `needs-review: true` | Overwrite with new version |
| `generated: true` + `needs-review: false` | Generate new version, compare. If different → report in Phase 5, do NOT overwrite |
| `generated: false` or missing flag | Do NOT touch. Report in Phase 5 if discovery suggests content is stale |
| New module found in codebase | Create new module document |
| Module removed from codebase | Do NOT delete. Report: "Module X not found, consider marking deprecated" |
| Manually added documents | Ignore completely — do not modify, do not delete |

**Error recovery:** If interrupted mid-generation, the user reruns `/init-docs`. Thanks to `generated: true` flags, the skill is idempotent — detects existing docs, fills gaps, updates incomplete ones.

---

## Documentation Principles

1. **No redundancy** — Don't describe what's visible in code (types, signatures). Describe intent and context.
2. **Conceptual focus** — How modules interact, their role in the system, architectural significance.
3. **Visual thinking** — Mermaid diagram for complex logic. Follow `${CLAUDE_SKILL_DIR}/references/conventions.md` guidelines (max 50 nodes, ELK for 20+, split strategy).
4. **Obsidian-native** — `[[wikilinks]]`, Dataview queries, Bases views where plugins are available. Always with plain Markdown fallback.
5. **One type per document** — Follow Diataxis (see `${CLAUDE_SKILL_DIR}/references/diataxis-guide.md`). `module` is a project extension of `reference`.
6. **Language follows user choice** — Content in chosen language. Frontmatter keys always English.
7. **Transparency** — Every AI-generated doc has `generated: true` + appropriate `needs-review` value.
8. **Adaptivity** — Use the project's own terminology. If codebase says "segment", docs say "segment". If structure doesn't match any known pattern, create an organization that reflects reality.
9. **Dual audience** — Purpose/Účel section of every module document must be readable by non-technical stakeholders (PO, PM, business owners). No class names, no code references, no technical jargon — business terminology only. Technical details (entities, services, implementations) belong in "How it works" and "Key components" sections, which target developers.