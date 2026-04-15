# Diataxis Documentation Framework — Quick Reference

## Table of Contents
- [Overview](#overview)
- [The Four Types](#the-four-types)
- [Module — Project Extension](#module--project-extension)
- [Classification Guide](#classification-guide)

## Overview

Diataxis organizes documentation into four types based on two axes:
- **Practical / Theoretical** (doing vs understanding)
- **Learning / Working** (studying vs applying)

|              | Learning         | Working          |
|--------------|------------------|------------------|
| **Practical**| Tutorial         | How-to Guide     |
| **Theoretical**| Explanation    | Reference        |

## The Four Types

### Tutorial
**"Learning by doing"** — Step-by-step guided experience for beginners.
- Teaches through a concrete project/task
- Author controls pace and scope
- Focuses on learning, not the end result
- **Skill generates:** NEVER (requires pedagogical design by humans)

### How-to Guide
**"Solving a specific problem"** — Practical steps for a known task.
- Assumes the reader already understands basics
- Goal-oriented: "How to deploy X", "How to add a new module"
- Reader knows what they want to do
- **Skill generates:** Only migrates existing guides from the codebase

### Reference
**"Information-oriented"** — Factual, structured description of the system.
- Consistent structure (like a dictionary or API docs)
- Accurate, complete, up-to-date
- No opinion, no tutorial steps
- **Skill generates:** Tech_Stack.md, Entity_Map.md, API_Endpoints.md, Module_Map.md, etc.

### Explanation
**"Understanding-oriented"** — Why things are the way they are.
- Big-picture context, design decisions, trade-offs
- Can be opinionated, discursive
- Answers "why" rather than "what" or "how"
- **Skill generates:** Only if strong context exists (CLAUDE.md, README, comments), always with `needs-review: true`

## Module — Project Extension

`type: module` is a project-specific extension of Reference. It documents one logical module/component/bundle of the codebase:
- Purpose and role in the system
- How it works (mechanism, not syntax)
- Key components with brief descriptions
- Dependencies (internal + external)
- Mermaid diagram if complex enough

Module docs are specialized reference documents — they describe "what is" per module. They do NOT teach (tutorial) or guide (how-to).

## Classification Guide

| Question | Type |
|----------|------|
| "Take me through building X step by step" | Tutorial |
| "How do I accomplish Y?" | How-to |
| "What is X and what does it contain?" | Reference / Module |
| "Why was X designed this way?" | Explanation |

**Key rule:** One type per document. Never mix tutorial steps into a reference, or explanation into a how-to.