---
name: Smart Commit
description: Analyse changes and create a conventional commit
allowed_tools:
  - Bash(git status)
  - Bash(git diff :*)
  - Bash(git log :*)
  - Bash(git add :*)
  - Bash(git commit :*)
argument-hint: "[ticket-id] [--no-body] [--split]"
---

# Smart Conventional Commit

Create a conventional commit for the current changes. Include ticket number $ARGUMENTS if provided.

## Rules

- Use the Conventional Commits spec: `<type>[(scope)]: <description>`
- Description must be lowercase, imperative mood, no trailing period, max 72 chars
- Body should explain *why*, not *what* — the diff already shows what
- Match the conventions visible in recent git history
- If `--no-body` is passed, omit the body
- If `--split` is passed and changes span unrelated concerns, create separate commits for each
- Body should not include a message to say the commit was co authored by Claude

## Behaviour

- If nothing is staged, stage all changes without asking
- If changes look suspicious (leaked secrets, debug logs, accidental files), warn before committing
- If the intent is ambiguous or changes span multiple unrelated concerns (and `--split` was not passed), use AskUserQuestionTool to clarify
- Show the final commit message and get confirmation before committing

## Context

! git diff --cached --stat
! git diff --cached
! git diff
! git status --short
! git log --oneline -15
