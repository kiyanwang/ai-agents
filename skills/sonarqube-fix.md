---
name: sonarqube-fix
description: Fix SonarQube issues on a GitHub pull request, review SonarQube bot comments, retrieve issue details from SonarCloud API, triage sonarqube findings, resolve sonarqube code smells, PR sonarqube review, sonarqube bot fix, automate sonarqube PR analysis, work through sonarqube issues on a pull request, sonarcloud quality gate failures
version: 0.1.0
---

# SonarQube PR Issue Resolver

Automatically detect SonarQube bot comments on a GitHub pull request, retrieve full issue details from the SonarCloud API, analyse each finding, and either fix it or explain why it's a false positive.

SonarQube bot comments only contain summary counts and links — they do not inline individual issue details. This skill bridges that gap by fetching the actual issues and security hotspots from the SonarCloud API.

## Input

The user provides a GitHub pull request URL or number plus repo. Examples:

- `https://github.com/org/repo/pull/123`
- `org/repo#123`
- `#123` (when inside a cloned repo)

**Argument**: `$ARGUMENTS` — the PR URL or identifier.

## Prerequisites

The SonarCloud API requires authentication. The token must be available as the environment variable `SONAR_TOKEN`. If it's not set, tell the user:

> Set your SonarCloud token: `export SONAR_TOKEN=<your-token>`
> Generate one at https://sonarcloud.io/account/security

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

### Step 2 — Find SonarQube bot comments

Fetch issue comments (not review comments) from the PR:

```bash
gh api "repos/{owner}/{repo}/issues/{number}/comments" --paginate -q '
  .[] | select(.user.login == "sonarqubecloud[bot]") |
  {id, body, html_url, created_at, updated_at}
'
```

If zero SonarQube comments are found, inform the user and stop.

### Step 3 — Extract project key and metadata from comment

Parse the SonarQube bot comment body to extract:

| Field | How to extract |
|---|---|
| **Quality Gate status** | Text after badge: `Quality Gate passed` or `Quality Gate failed` |
| **Project key** | From any `sonarcloud.io` URL in the body: the `id=` query parameter |
| **PR number** | From any `sonarcloud.io` URL in the body: the `pullRequest=` query parameter |
| **Issue count** | Number before "New issues" in the failed conditions |
| **Hotspot count** | Number before "Security Hotspots" in the failed conditions |

If the Quality Gate passed and there are no issues or hotspots, inform the user and stop.

### Step 4 — Fetch issue details from SonarCloud API

Use the extracted project key and PR number to call the SonarCloud API. Make both calls in parallel:

**Fetch issues:**
```bash
curl -s -H "Authorization: Bearer $SONAR_TOKEN" \
  "https://sonarcloud.io/api/issues/search?componentKeys={projectKey}&pullRequest={prNumber}&issueStatuses=OPEN,CONFIRMED&sinceLeakPeriod=true&ps=100"
```

**Fetch security hotspots:**
```bash
curl -s -H "Authorization: Bearer $SONAR_TOKEN" \
  "https://sonarcloud.io/api/hotspots/search?projectKey={projectKey}&pullRequest={prNumber}&ps=100"
```

For each issue returned, extract:

| Field | JSON path |
|---|---|
| **Key** | `.issues[].key` |
| **Rule** | `.issues[].rule` |
| **Severity** | `.issues[].severity` |
| **Type** | `.issues[].type` (BUG, VULNERABILITY, CODE_SMELL) |
| **Message** | `.issues[].message` |
| **Component** | `.issues[].component` (full path) |
| **File path** | Strip the project prefix from component to get relative path |
| **Line** | `.issues[].line` |
| **Text range** | `.issues[].textRange` (startLine, endLine, startOffset, endOffset) |
| **Effort** | `.issues[].effort` |
| **Tags** | `.issues[].tags` |

For each hotspot returned, extract:

| Field | JSON path |
|---|---|
| **Key** | `.hotspots[].key` |
| **Rule** | `.hotspots[].securityCategory` |
| **Status** | `.hotspots[].status` |
| **Message** | `.hotspots[].message` |
| **Component** | `.hotspots[].component` |
| **File path** | Strip the project prefix from component to get relative path |
| **Line** | `.hotspots[].line` |
| **Vulnerability probability** | `.hotspots[].vulnerabilityProbability` |

### Step 5 — Fetch rule details for context

For each unique rule ID found across issues and hotspots, fetch the rule description:

```bash
curl -s -H "Authorization: Bearer $SONAR_TOKEN" \
  "https://sonarcloud.io/api/rules/show?key={ruleKey}"
```

Extract the rule's `name`, `htmlDesc` (or `mdDesc`), and `type` to understand what the rule checks. Batch these requests — do not fetch the same rule twice.

### Step 6 — Present findings summary

Present a numbered summary table to the user, grouping issues and hotspots separately:

```
## Issues (3)

# | Severity | Type       | File                  | Line | Message
--|----------|------------|-----------------------|------|--------
1 | MAJOR    | CODE_SMELL | src/auth/login.ts     | 42   | Remove this unused import
2 | CRITICAL | BUG        | src/users/service.ts  | 189  | This condition always evaluates to true
3 | MAJOR    | VULNERABILITY | src/api/handler.ts | 67   | Use a secure random generator

## Security Hotspots (2)

# | Probability | Category          | File                | Line | Message
--|-------------|-------------------|---------------------|------|--------
4 | HIGH        | sql-injection     | src/db/queries.ts   | 34   | Make sure this SQL query is safe
5 | MEDIUM      | weak-cryptography | src/crypto/hash.ts  | 12   | Use a stronger hashing algorithm
```

### Step 7 — Work through each finding

Process findings in this priority order:
1. Issues with severity BLOCKER or CRITICAL
2. Security Hotspots with HIGH vulnerability probability
3. Issues with severity MAJOR
4. Security Hotspots with MEDIUM vulnerability probability
5. Remaining issues and hotspots (LOW, INFO)

For each finding:

1. **Observe** — Read the identified file and line range. Include sufficient surrounding context. State what the code does.
2. **Assess** — Consider the rule description, the code context, and the specific message. Determine whether the finding is:
   - **Valid** — the code genuinely has this problem
   - **Partially valid** — the finding has merit but the suggested approach needs adjustment
   - **False positive** — the code is correct; the static analysis is wrong here
   State reasoning clearly.
3. **Act** — Based on the assessment:

| Assessment | Action |
|---|---|
| Valid | Implement the fix. Show the change to the user. |
| Partially valid | Implement what's needed. Explain what was and wasn't applicable. |
| False positive | Explain why. Do not modify code. |

When implementing a fix:
- Read enough surrounding context to understand the code.
- Follow existing code style and patterns in the file.
- Make minimal, targeted changes — do not refactor unrelated code.
- If a fix requires changes outside the PR's scope (e.g., a database migration, config change), describe what's needed rather than guessing.

### Step 8 — Summarise

After processing all findings, output a summary:

```
## SonarQube Findings Summary

| # | Severity | Type | File | Verdict | Action taken |
|---|----------|------|------|---------|--------------|
| 1 | MAJOR | CODE_SMELL | src/auth/login.ts | Valid | Fixed — removed unused import |
| 2 | CRITICAL | BUG | src/users/service.ts | False positive | No change — condition is intentional guard |
| 3 | HIGH hotspot | sql-injection | src/db/queries.ts | Valid | Fixed — parameterised query |

### Changes made
- `src/auth/login.ts`: removed unused import of `debugLog`
- `src/db/queries.ts`: replaced string interpolation with parameterised query

### Remaining items requiring manual attention
- Issue #4: Requires migration to update password hashing algorithm — outside PR scope
```

## Constraints

- Never dismiss a CRITICAL or BLOCKER severity finding without thorough verification.
- Never dismiss a HIGH probability security hotspot without thorough verification.
- Never modify files outside the scope of the SonarQube findings unless necessary for the fix.
- Always show reasoning before acting — no silent fixes.
- If the repository is not cloned locally, use `gh` commands to read file contents rather than asking the user to clone.
- If multiple findings affect the same file, batch-read the file once and process all findings together.
- Rate limit API calls — do not make redundant requests for the same rule or endpoint.

## Error handling

| Problem | Response |
|---|---|
| `gh` not authenticated | Tell the user to run `gh auth login` |
| `SONAR_TOKEN` not set or API returns 401 | Tell the user to set `SONAR_TOKEN` and provide the generation URL |
| PR not found | Ask user to verify the URL and their access |
| No SonarQube comments | Inform user: "No SonarQube bot comments found on this PR." |
| Quality Gate passed with no issues | Inform user: "Quality Gate passed — no issues to review." |
| SonarCloud API returns empty issues | Check if the PR analysis has completed; inform user if results are pending |
| File referenced by SonarQube doesn't exist locally | Use `gh api` to fetch file contents from the PR branch |
| API rate limiting (429) | Wait and retry once; if still limited, inform the user |

<example>
**User request:**
```
/sonarqube-fix https://github.com/biblio-tech/platform/pull/613
```

**Agent behaviour:**

1. Parses → owner: `biblio-tech`, repo: `platform`, PR: `613`
2. Fetches issue comments, filters to `sonarqubecloud[bot]`
3. Finds 1 comment with Quality Gate failed:
   - Project key: `biblio-tech_platform`
   - 3 New issues, 11 Security Hotspots
4. Calls SonarCloud API to fetch all issues and hotspots for PR 613
5. Fetches rule descriptions for each unique rule
6. Presents summary table:
   ```
   ## Issues (3)

   # | Severity | Type       | File                          | Line | Message
   --|----------|------------|-------------------------------|------|--------
   1 | MAJOR    | CODE_SMELL | src/auth/wayf/handler.ts      | 45   | Cognitive complexity is too high
   2 | MAJOR    | BUG        | src/auth/wayf/institution.ts  | 112  | This always returns null
   3 | MINOR    | CODE_SMELL | src/auth/wayf/types.ts        | 8    | Remove unused type export

   ## Security Hotspots (11)
   ...
   ```
7. Works through each finding:
   - Reads the file and relevant lines
   - States observation, assessment, and action
   - Implements fixes for valid findings
8. Outputs final summary with verdicts and changes made
</example>
