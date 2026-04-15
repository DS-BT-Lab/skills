---
type: module
status: <<status>>
date: <<date>>
generated: true
needs-review: true
tags: [type/module, domain/<<domain>>, layer/<<layer>>]
aliases: [<<aliases>>]
depends-on-external: [<<external_deps>>]
---

## <<section_purpose>>

<<2-3 sentences describing the module's role in business terms. Must be understandable without code knowledge — no class names, no framework terms, no technical jargon. Focus on WHAT it does for the business and WHY it exists.>>

## <<section_how_it_works>>

<<Brief description of the module's mechanism and its interactions with other modules. Describe the flow, not the code.>>

## <<section_key_components>>

| <<col_component>> | <<col_role>> |
|---|---|
| <<component_name>> | <<component_description>> |

## <<section_dependencies>>

**<<label_internal>>:**
- [[ModuleA]] — <<relationship_description>>

**<<label_external>>:**
- <<external_dependency>> — <<why_needed>>

## <<section_diagram>>

> Include Mermaid diagram only if module has sufficient complexity (3+ entities/components).
> Follow Mermaid conventions from references/conventions.md.

```mermaid
<<diagram_content>>
```
