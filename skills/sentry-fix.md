---
name: sentry-fix
description: Fix Sentry issues on a GitHub pull request, review Sentry bot comments, extract Prompt for AI Agent, triage sentry bugs, resolve sentry findings, PR sentry review, sentry bot fix, automate sentry PR comments, work through sentry issues on a pull request
version: 0.1.0
---

# Sentry PR Issue Resolver

Automatically extract and resolve Sentry bot findings on a GitHub pull request. Read each Sentry comment, extract the "Prompt for AI Agent" block, verify whether the issue is real, and either fix it or explain why it's a false positive.

## Input

The user provides a GitHub pull request URL or number plus repo. Examples:

- `https://github.com/org/repo/pull/123`
- `org/repo#123`
- `#123` (when inside a cloned repo)

**Argument**: `$ARGUMENTS` — the PR URL or identifier.

## Workflow

Follow these steps in strict order.

### Step 1 — Parse the PR identifier

Extract the owner, repo, and PR number from `$ARGUMENTS`.

| Format | Example | Parse rule |
|---|---|---|
| Full URL | `https://github.com/org/repo/pull/123` | Split on `/` |
| Short ref | `org/repo#123` | Split on `/` and `#` |
| Number only | `#123` or `123` | Use current repo from `gh repo view --json nameWithOwner -q .nameWithOwner` |

If parsing fails, stop and ask the user for a valid PR reference.

### Step 2 — Fetch Sentry comments

Run:

```bash
gh api "repos/{owner}/{repo}/pulls/{number}/comments" --paginate -q '
  .[] | select(.user.login == "sentry[bot]") |
  {id, path, line, start_line, body, html_url}
'
```

If zero Sentry comments are found, inform the user and stop.

### Step 3 — Extract and triage each finding

For each Sentry comment, extract these fields from the comment body:

| Field | How to extract |
|---|---|
| **Severity** | Text inside `<sub>Severity: ...</sub>` |
| **Bug summary** | Text after `**Bug:**` up to the severity tag |
| **Suggested fix** | Content inside the `<details>` block whose `<summary>` contains "Suggested Fix" |
| **AI agent prompt** | Content of the fenced code block inside the `<details>` block whose `<summary>` contains "Prompt for AI Agent" |
| **File path** | The `path` field from the API response |
| **Line range** | The `Location:` line inside the AI agent prompt, or fall back to `line`/`start_line` from the API |
| **Reference ID** | The number inside `<b title="Reference ID: ...">` |

Present a numbered summary table to the user:

```
# | Severity | File | Lines | Bug summary
--|----------|------|-------|------------
1 | HIGH     | src/users.ts | 289-308 | Missing UNIQUE constraint ...
```

### Step 4 — Work through each finding

Process findings from highest severity to lowest. For each finding:

1. **Observe** — Read the identified file and line range. State what the code does.
2. **Conclude** — Determine whether the Sentry finding is valid, a false positive, or partially valid. State reasoning.
3. **Act** — Based on the conclusion:

| Conclusion | Action |
|---|---|
| Valid bug | Implement the fix. Show the change to the user. |
| Partially valid | Implement what's needed. Explain what was and wasn't applicable. |
| False positive | Explain why. Do not modify code. |

When implementing a fix:
- Read enough surrounding context to understand the code.
- Follow existing code style and patterns in the file.
- Make minimal, targeted changes — do not refactor unrelated code.
- If a fix requires changes outside the PR's scope (e.g., a database migration), describe what's needed rather than guessing at the project's migration tooling.

### Step 5 — Summarise

After processing all findings, output a summary:

```
## Sentry Findings Summary

| # | Severity | File | Verdict | Action taken |
|---|----------|------|---------|--------------|
| 1 | HIGH | src/users.ts | Valid | Fixed — added UNIQUE constraint migration |
| 2 | MEDIUM | src/auth.ts | False positive | No change — constraint exists in migration v42 |

### Changes made
- `src/users.ts`: <one-line description>
- `migrations/005_add_unique.sql`: <one-line description>
```

## Constraints

- Never dismiss a HIGH severity finding without thorough verification.
- Never modify files outside the scope of the Sentry findings unless necessary for the fix.
- Always show reasoning before acting — no silent fixes.
- If the repository is not cloned locally, use `gh` commands to read file contents rather than asking the user to clone.
- If multiple findings affect the same file, batch-read the file once and process all findings together.

## Error handling

| Problem | Response |
|---|---|
| `gh` not authenticated | Tell the user to run `gh auth login` |
| PR not found | Ask user to verify the URL and their access |
| No Sentry comments | Inform user: "No Sentry bot comments found on this PR." |
| File referenced by Sentry doesn't exist locally | Use `gh api` to fetch file contents from the PR branch |

<example>
**User request:**
```
/sentry-fix https://github.com/biblio-tech/institutions/pull/10963
```

**Agent behaviour:**

1. Parses → owner: `biblio-tech`, repo: `institutions`, PR: `10963`
2. Fetches review comments, filters to `sentry[bot]`
3. Finds 1 comment:
   ```
   # | Severity | File | Lines | Bug summary
   --|----------|------|-------|------------
   1 | HIGH     | packages/bibliu/src/institutions/users.ts | 289-308 | Missing UNIQUE(email, InstitutionId) constraint
   ```
4. Reads `packages/bibliu/src/institutions/users.ts` lines 273–310
5. States observation: "The `create()` and `upsert()` methods use `ON DUPLICATE KEY UPDATE` which requires a UNIQUE constraint on (email, InstitutionId)."
6. States conclusion: "Valid — no migration adding this constraint exists in the PR."
7. States action: "This requires a database migration. Describing the needed migration rather than generating one, since the project's migration tooling is unknown."
8. Outputs summary table.
</example>
