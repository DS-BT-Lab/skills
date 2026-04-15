# Scope Decomposition Reference

Reference heuristics for Phase 0.3 of the doc-full-research orchestrator.
Used to decompose project modules into ordered research units.

---

## 1. Dependency Detection Patterns

For each module listed in `project-profile.yaml`, scan its source files for
import/reference patterns pointing to OTHER modules' paths. Build an adjacency
map: `module_name -> [depends_on_module_1, ...]`.

Tech stack comes from `project-profile.yaml` field `techStack[].language`.
Select the matching pattern set:

| Language | Import pattern | Grep regex |
|---|---|---|
| PHP | `use App\{ModulePath}\...` | `^use\s+App\\\\` |
| TypeScript / JavaScript | `import ... from '{modulePath}'` | `from\s+['"]\.*/` |
| Python | `from {package} import ...` | `^from\s+\w+\s+import` |
| Go | `import "{module}/...` | `import\s+"` |

**Procedure:**
1. For each module, collect all source files under its `path` from the profile.
2. Grep those files using the language-appropriate regex.
3. Match captured import paths against other modules' `path` values.
4. Record each match as a directed edge: `this_module -> imported_module`.

These patterns tell WHERE to look for cross-module references, not HOW to
interpret business logic. The result is a raw adjacency map for topological sort.

---

## 2. Topological Sort Algorithm

Order modules so dependencies come before dependents (leaf-first).

```
function topologicalSort(modules, adjacencyMap):
    inDegree = {}
    for each module in modules:
        inDegree[module] = 0

    for each module, dependencies in adjacencyMap:
        for each dep in dependencies:
            inDegree[dep] += 1

    queue = [m for m in modules if inDegree[m] == 0]
    ordered = []

    while queue is not empty:
        current = queue.pop(0)
        ordered.append(current)
        for each dependent where current in adjacencyMap[dependent]:
            inDegree[dependent] -= 1
            if inDegree[dependent] == 0:
                queue.append(dependent)

    if len(ordered) < len(modules):
        # Cycle detected
        remaining = modules - ordered
        breakCycle(remaining, adjacencyMap)

    return ordered
```

**Cycle-breaking strategy:**
1. Identify strongly connected components among remaining modules.
2. Within each component, select the module with the fewest total dependencies
   (sum of in-edges + out-edges) as the entry point.
3. Place that module first, then re-run sort on the rest of the component.
4. Mark the cycle in output so the orchestrator can note it in research units.

**Output:** Ordered list of module names, leaf modules first.

---

## 3. Grouping Heuristics

After topological ordering, group modules into research units. Apply heuristics
in this priority order:

1. **Domain coupling** -- Modules sharing entity/model relationships. Detected
   from the adjacency map: mutual dependencies or multiple modules with heavy
   shared dependencies on the same third module. These belong together.

2. **Type matching** -- Modules with the same `type` field value from the
   project profile (e.g., two "feature" modules or two "infrastructure" modules).
   Same-type modules are natural candidates for joint analysis.

3. **Functional proximity** -- Modules in adjacent topological positions that
   share boundary modules (common dependencies or common dependents). Adjacency
   in the sort order plus shared edges signals tight functional relationship.

4. **Size balancing** -- Prevent units from being too large or too small:
   - Large module (main source >30KB or >10 key components listed in profile):
     gets its own dedicated research unit.
   - Medium module (5-10 components): group 2-3 modules per unit.
   - Small module (<5 components): group 3-5 modules per unit.

**Constraint:** Grouping MUST respect topological order. All dependencies of
modules in a group must appear in earlier groups or within the same group. Never
place a dependency in a later group than its dependent.

---

## 4. Sizing Thresholds

Target number of research units based on total project module count:

| Project modules | Target units | Modules per unit |
|---|---|---|
| 3-8 | 2-3 | 2-4 |
| 9-20 | 4-7 | 2-5 |
| 21-50 | 6-10 | 3-7 |
| 50+ | Ask user to select scope | -- |

For 50+ modules, prompt the user to select a subset of modules or a domain area
before proceeding. Full-project research at that scale requires explicit scoping.

---

## 5. Topic Formulation Templates

Each research unit needs a topic string for the subagent prompt.

**Single-module unit:**
`"Deep-dive into {module.name} -- {aspects from module type and tech stack}"`

**Multi-module unit:**
`"Deep-dive into {module1.name}, {module2.name}, {module3.name} -- {shared aspect from group type/domain}"`

Aspects are selected based on the module `type` field from the profile:

| Module type | Aspects |
|---|---|
| feature | business logic, entity relationships, workflows |
| bundle | service registration, extension points, configuration |
| infrastructure | adapters, integrations, configuration |
| shared | cross-cutting patterns, reuse points |

When a unit contains modules of mixed types, combine the relevant aspect sets.
For example, a unit with one "feature" and one "infrastructure" module uses:
`"business logic, entity relationships, integrations"`.

---

## 6. Boundary Context Templates

Each research unit receives a scope description that defines what to analyze
deeply and what to reference only for context.

Template:

```
Focus scope: Deeply analyze {module list with paths from profile}.
Boundary modules (reference but do not deep-dive):
{For each adjacent module from dependency map:
  - {module.name} ({module.path}) -- {relationship type} dependency}
Already-documented dependencies (read their vault docs for context):
{For each dependency module already processed in earlier units:
  - {vault doc path} -- documented in previous research unit}
```

Relationship types for boundary modules:
- "upstream" -- the boundary module depends on our focus module
- "downstream" -- our focus module depends on the boundary module
- "mutual" -- bidirectional dependency (cycle marker)

Already-documented modules should reference their vault doc paths so the
subagent can read existing documentation rather than re-analyzing source code.

---

## 7. Skip Detection Criteria

Before assigning a module to a research unit, check if adequate vault
documentation already exists. Classify existing docs into three tiers:

**Deep (skip research):**
- 100+ content lines (excluding YAML frontmatter)
- Contains at least one Mermaid diagram
- Has detailed component sections (3+ individual component entries)
- Frontmatter field `needs-review: false`
- All four conditions must be met to classify as deep.

**Shallow (needs full research):**
- <30 content lines
- No Mermaid diagram present
- Only Purpose/overview section filled, no component detail
- Any one condition is sufficient to classify as shallow.

**Moderate (needs targeted update):**
- Between shallow and deep thresholds
- Has some component detail but missing diagrams or depth
- Research subagent should focus on gaps rather than full re-analysis

Modules with deep documentation are excluded from research units entirely.
Modules with moderate documentation are included but flagged so the subagent
knows to update rather than write from scratch.
