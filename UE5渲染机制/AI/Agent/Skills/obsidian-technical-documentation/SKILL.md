---
name: obsidian-technical-documentation
description: Create or revise reader-first, evidence-grounded technical notes in an Obsidian Vault. Use when documenting a system, module, architecture, workflow, design decision, research conclusion, or README from code or experiments, especially when the Markdown must stay accurate, navigable, and safe to commit.
---

# Obsidian Technical Documentation

Turn verified technical material into a note that lets a future reader understand what exists, why it matters, and where uncertainty remains. Prefer durable explanations over a directory listing or an API dump.

## Choose the document mode

Infer the reader and mode from the request. Ask one focused question only when the choice would materially change the document.

| Mode | Reader's first need | Default shape |
|---|---|---|
| Orientation / README | What is this and how do its parts fit? | Purpose, scope, map, boundaries, next documents |
| Module explanation | How does this work? | Mental model, flow, decisions, edge cases |
| Decision record | Why was this choice made? | Context, decision, alternatives, consequences |
| Research note | What does the evidence support? | Question, method, evidence, limits, next test |
| Runbook | What must I safely do? | Preconditions, steps, verification, rollback/escalation |

Write for the least-specialized intended reader. Lead with the outcome and introduce a concept before relying on it.

## Establish facts before prose

1. Read the target note and its immediate links to preserve the Vault's conventions.
2. Use the current source of truth: code and configuration for implementation facts; experiment artifacts for measured claims; the user's stated decisions for intent. Do not treat stale prose as evidence.
3. Separate facts, interpretation, and unknowns. Use wording such as “the current implementation exposes…”, “the evidence suggests…”, or “not yet verified” rather than filling gaps with certainty.
4. Do not make code paths the main content. Mention a path, API, or command only when it lets the reader verify or perform a necessary action.

Never put credentials, private keys, API secrets, account exports, or sensitive local configuration in the note.

## Draft the smallest useful structure

Start with a title and one paragraph that answers the reader's first question. Add only sections that earn their place.

- Use headings that communicate information, not generic labels such as “Overview” or “Details”.
- Give one durable idea per section and place parallel information in lists or tables.
- Use a concrete example only when it makes an abstract mechanism easier to apply.
- State exclusions and limitations close to the claim they qualify.
- Link to a dedicated note instead of duplicating detailed material.

For architecture or module notes, explain the relationship before the implementation detail:

```text
intent → responsibilities → data/event flow → decisions and boundaries
```

For a decision, record rejected alternatives and consequences, not a restatement of the chosen code.

## Use visuals deliberately

Add a small Mermaid, ASCII flow, table, or sequence only when it makes a relationship materially clearer than prose. Typical triggers are three or more components, a lifecycle with state changes, or repeated field mappings.

- Use a flow/sequence for time or data movement.
- Use a table for parallel comparisons or mappings.
- Use a tree for ownership or nesting.
- Keep diagrams focused on the reader's question; do not decorate a simple note.

## Edit and review safely

When the note lives in an Obsidian Git Vault, follow the `obsidian-git-workflow` safeguards: inspect the target and Git state before editing, preserve unrelated changes, use precise edits, and commit or push only with explicit authorization.

Before handoff, check:

- The opening states what the reader gains from the note.
- Claims have a current evidence source or a visible uncertainty label.
- The structure matches the document mode and contains no filler sections.
- Each visual earns its space and agrees with the prose.
- Links, terminology, and headings are easy to scan.
- No credentials or accidental local state were added.
- The final diff contains only the requested note changes.

For a README, use `create-readme` when a concise project-facing presentation is the goal. For durable architectural choices, use `documentation-and-adrs` to capture context, alternatives, and consequences. Use this skill as the integration layer that keeps the resulting note reader-first and Vault-safe.
