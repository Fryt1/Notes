---
name: obsidian-git-workflow
description: Safely write and revise Markdown notes in an Obsidian Vault and manage their Git changes. Use when a user asks to create, organize, rewrite, link, commit, push, or back up Obsidian notes stored in a Git repository.
---

# Obsidian Git Workflow

Treat the Vault as the source of truth: notes are Markdown files and Git records their history. Preserve the user's writing and repository state; do not treat a Vault as an empty documentation project.

## 1. Discover the scope first

1. Use the Vault path supplied by the user. If it is absent, read Obsidian's vault registry only when it is available locally, then ask the user to confirm the intended Vault when several candidates exist.
2. Resolve the Git root, current branch, remotes, and `git status --short`.
3. Report pre-existing modified, deleted, or untracked files before writing or staging anything. Never assume they belong to the request.
4. Do not create a new Vault, change a remote, install a plugin, expose a NAS, or create cloud credentials unless the user explicitly asks.

## 2. Write Obsidian notes

- Read the target note and closely related notes before editing. Preserve the existing language, headings, frontmatter, tags, links, and folder convention.
- Put one durable idea per note. Use a clear H1 title when the Vault convention calls for it; do not add frontmatter, daily-note metadata, or tags merely because they are possible.
- Prefer relative wiki links (`[[Note Name]]`) or Markdown links that match existing Vault style. Verify renamed or moved notes do not leave broken links.
- Put attachments in the Vault's existing attachment folder. Do not add large datasets, generated build output, credentials, or exchange/API exports to the Vault repository.
- Treat `.obsidian/workspace*.json`, caches, trash folders, and per-device API configuration as local state. Do not edit or stage them unless the user specifically asks.
- Never print secrets found in plugin settings, Git credentials, `.env` files, or API configuration.

## 3. Review changes before Git actions

After edits, show a concise summary and inspect the exact diff:

```text
git status --short
git diff -- <selected paths>
```

Keep the user's unrelated changes unstaged. Do not use these commands in a Vault:

```text
git add -A
git add .
git commit -a
git reset --hard
git checkout -- .
git clean -fd
git push --force
```

Stage only exact, user-requested Markdown or configuration paths:

```text
git add -- "path/to/note.md" "path/to/another-note.md"
git diff --cached --stat
git diff --cached -- "path/to/note.md"
```

If `.gitignore` is missing or local Obsidian state is tracked, propose a minimal rule first. Removing an already tracked local-state file requires `git rm --cached`; explain that it preserves the local file and obtain approval before doing it.

## 4. Commit and push require explicit authorization

Interpret user intent narrowly:

| User asks for | Allowed action |
| --- | --- |
| "写/改笔记" | Edit notes only; do not stage, commit, or push. |
| "准备提交" | Review and stage selected files; do not commit. |
| "提交" | Commit only the reviewed staged files; do not push. |
| "推送/同步到 GitHub" | Check divergence, then push the reviewed commit to the stated remote and branch. |

Before committing, state the staged paths and commit message. Before pushing, state the remote and branch, then inspect whether the upstream has diverged. If it has diverged, stop and ask rather than automatically pulling, rebasing, merging, or force-pushing.

Prefer a feature branch and pull request for agent-authored changes. A direct push to the current branch is allowed only when the user explicitly requests it and the staged diff contains only the agreed paths.

## 5. NAS and cloud backup boundaries

- Do not use a network share as a simultaneously-open Vault across devices. Keep a local working copy per device and synchronize deliberately.
- Git protects Markdown history but is not a complete backup. Recommend encrypted, versioned backup for attachments, Vault settings, and NAS configuration.
- Keep cloud credentials out of the Vault. Use environment variables, a credential manager, or NAS-managed secrets.

## Completion report

Report the Vault path, changed note paths, whether changes are only local/staged/committed/pushed, the commit hash when created, and the remote/branch when pushed. Call out any unrelated dirty files left untouched.
