---
name: doc-generator
description: "Generate or revise code-adjacent documentation: Python docstrings, JSDoc/TSDoc, class and module comments, API endpoint references, and changelog fragments. Use when documentation must be derived from the implementation. Do not use for READMEs, architecture documents, research notes, or Obsidian technical notes."
---

# Code Documentation Generator

Keep documentation close to the implementation and grounded in what the code actually does.

## Workflow

1. Read the target code, its types or schemas, and relevant tests before writing.
2. Extract only supported facts: purpose, inputs, outputs, side effects, error conditions, API request and response shapes, and compatibility constraints.
3. Use the established format in the repository. Prefer concise Python docstrings or JSDoc/TSDoc; use endpoint-reference Markdown only when an API surface needs it.
4. Write an example only when it is supported by code or tests. Do not invent exceptions, defaults, HTTP responses, or behavior.
5. Re-read the edited section and check that names, types, and claims agree with the current implementation.

## Boundaries

- Use `create-readme` for project-facing READMEs.
- Use `obsidian-technical-documentation` for reader-first architecture, workflow, research, and Vault notes.
- Use `documentation-and-adrs` for durable decisions and ADRs.
- Keep changelog fragments factual and limited to user-visible changes already made.
