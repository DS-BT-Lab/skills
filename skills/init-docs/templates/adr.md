---
type: adr
status: proposed
date: <<date>>
generated: true
needs-review: true
tags: [type/adr, domain/<<domain>>]
supersedes: <<supersedes>>
superseded-by: <<superseded_by>>
---

# <<adr_number>>: <<adr_title>>

## <<section_context_and_problem>>

<<Situation, forces, and constraints that led to this decision.>>

## <<section_considered_options>>

1. **<<option_a>>** — <<brief_description>>
2. **<<option_b>>** — <<brief_description>>

## <<section_decision>>

<<Chosen option + rationale: "We chose X because Y.">>

> **Anti-hallucination rule:** If the business reason is not known from code or documentation, write:
> "Detected pattern X. Business reason is unknown — please supplement with context."
> The Decision section must then contain only a factual description of WHAT is in the code,
> never an invented justification.

## <<section_consequences>>

- **<<label_easier>>:** <<positive_consequence>>
- **<<label_harder>>:** <<negative_consequence_or_tradeoff>>

---

**Status workflow:** `proposed` → `accepted` | `rejected`, then `accepted` → `deprecated` | `superseded by ADR-XXXX`
