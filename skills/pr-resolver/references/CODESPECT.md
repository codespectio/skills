# Bot Comment Parsing Guide

This reference covers how to extract actionable information from automated review bot comments. Load this reference whenever a PR has comments from `codespect-io[bot]` or `coderabbitai[bot]`.

## Codespect-io Bot

Bot identifier: `user.login === "codespect-io[bot]"` and `user.type === "Bot"`

### Review Summary (Top-Level Review Body)

Codespect posts a single review with a summary body and multiple inline comments. The review body follows this pattern:

```
This PR contains: 3 minor issues and 1 nitpick. <Overall assessment text>
```

**Parsing the severity counts:**
- Look for pattern: `(\d+)\s+minor\s+issue` → extract minor count
- Look for pattern: `(\d+)\s+nitpick` → extract nitpick count
- Look for pattern: `(\d+)\s+critical` or `(\d+)\s+major` → extract critical/major count (rare)
- These counts help you verify your individual comment classification

### PR Summary Comment (Issue Comment)

Codespect also posts a general PR comment (not a review comment) with a structured summary. This appears as an issue comment (`/issues/comments` endpoint) rather than a review comment. It contains:

```markdown
## Summary by CodeSpect

Features:
- <description>

Feature Improvement:
- <description>

Tests:
- <description>

## Overview
<details>
<summary>High level overview</summary>
| Key Change | Files Impacted |
|---|---|
| ... | ... |
</details>
```

This summary is informational — no action needed. Do not treat it as a review comment to fix.

### Individual Comment Structure

Each inline review comment from codespect follows this exact structure:

````markdown
**<Issue Title>**

<Explanation paragraph describing the problem and why it matters>

<details>
<summary>Code Suggestion</summary>

```suggestion
<suggested code replacement>
```
</details>

<details>
<summary>AI Fix Prompt</summary>

> Copy-paste this prompt into your AI assistant to get a fix for this issue.

```text
I have a code review comment on my pull request that I need to address.

File: `<file_path>` (line <line_number>)

Reviewer's comment:
**<Issue Title>**

<Explanation>

Suggested fix:
<code>

Please update the code in this file to address the reviewer's feedback. Show the corrected code and briefly explain what was changed.
```

</details>
````

### Extracting the AI Fix Prompt

This is the most valuable part for pr-resolver. The AI Fix Prompt contains a self-contained instruction that includes:
- The exact file path and line number
- The reviewer's comment in full
- A suggested code fix
- Clear instructions for what to do

**Extraction steps:**
1. Find the `<details>` block whose `<summary>` contains "AI Fix Prompt"
2. Inside that block, find the code fence with language `text`
3. Extract everything between the opening and closing code fence markers
4. This extracted prompt is your primary fix guidance — it contains all context needed

**Regex pattern for extraction:**
```
<details>\s*<summary>.*?AI Fix Prompt.*?</summary>\s*>.*?\n\s*```text\s*\n([\s\S]*?)```\s*</details>
```

### Extracting the Code Suggestion

The Code Suggestion is a direct code replacement that can be applied to the referenced lines.

**Extraction steps:**
1. Find the `<details>` block whose `<summary>` contains "Code Suggestion"
2. Inside that block, find the code fence with language `suggestion`
3. Extract the code content

**Regex pattern:**
```
<details>\s*<summary>.*?Code Suggestion.*?</summary>\s*```suggestion\s*\n([\s\S]*?)```\s*</details>
```

**Important:** The suggestion replaces the lines from `start_line` to `line` in the comment's API response. These fields tell you exactly which lines to replace:
- `start_line` → first line of the replacement range
- `line` → last line of the replacement range
- `path` → the file path relative to repo root

### Severity Detection for Codespect Comments

Codespect does NOT include per-comment severity labels in the body text. Severity is determined by:

1. **Title prefix**: If the bold title starts with `Nitpick:` → severity is `nitpick`
2. **Review summary context**: Cross-reference with the review summary counts
3. **Default**: If no prefix and not nitpick → severity is `minor`
4. **Escalation indicators**: If the explanation mentions "security", "data loss", "breaking change", or "architectural" → treat as `major` regardless

---

## CodeRabbit Bot

Bot identifier: `user.login === "coderabbitai[bot]"` and `user.type === "Bot"`

### Comment Structure

CodeRabbit comments have a different format with explicit severity labels:

````markdown
_<category>_ | _<severity>_

**<Title or explanation>**

<details>
<summary>Proposed fix using <approach></summary>

```diff
<diff content>
```
</details>

<details>
<summary>Committable suggestion</summary>

> IMPORTANT
> Carefully review the code before committing.

```suggestion
<code>
```
</details>

<details>
<summary>Prompt for AI Agents</summary>

```
<structured prompt for AI agents>
```
</details>
````

### Severity Labels

CodeRabbit includes explicit severity in the first line of each comment:

| Label | Severity | Auto-fix? |
|-------|----------|-----------|
| `_🟢 Info_` | Informational | Skip — no fix needed |
| `_🟡 Minor_` | Minor | Yes |
| `_🔴 Critical_` | Critical | Ask user (even in auto mode) |
| `_💡 Suggestion_` | Suggestion | Yes |

**Category labels** (appear before the severity):
- `_⚠️ Potential issue_` → likely needs a fix
- `_🧹 Nitpick_` → minor style/preference → auto-fix

**Regex for severity extraction:**
```
_([^_]+)_\s*\|\s*_([^_]+)_
```
Group 1 = category, Group 2 = severity (with emoji)

### Extracting the AI Agent Prompt

1. Find `<details>` with summary containing "Prompt for AI Agents"
2. Extract the code block content inside

**Regex:**
```
<details>\s*<summary>.*?Prompt for AI Agents.*?</summary>\s*```\s*\n([\s\S]*?)```\s*</details>
```

### Extracting Committable Suggestions

1. Find `<details>` with summary containing "Committable suggestion"
2. Extract the `suggestion` code block

**Regex:**
```
<details>\s*<summary>.*?Committable suggestion.*?</summary>[\s\S]*?```suggestion\s*\n([\s\S]*?)```\s*</details>
```

### CodeRabbit Review Summary Comment

CodeRabbit posts a top-level issue comment (not a review comment) with:
- Walkthrough with a table of all changes
- Sequence diagrams
- Review effort estimate
- Nitpick comments section (these are suggestions, not blocking)
- `Actionable comments posted: N` — this tells you how many inline comments need attention

The nitpick comments inside the summary are organized under `<details>` with `<summary>🧹 Nitpick comments (N)</summary>`. These are also inline review comments but grouped in the summary for convenience. The actual inline comments on the diff are the primary target.

---

## Generic Bot Detection

For any other bot reviewer (not codespect or coderabbit):

1. Check `user.type === "Bot"` in the API response
2. Look for common patterns in the body:
   - Code blocks with suggestions
   - `<details>` sections with fixes
   - Severity indicators in text (error, warning, info)
3. Fall back to treating the entire comment body as the review feedback
4. Use AI judgment to determine the fix

---

## Comment Resolution Detection

A comment should be considered **already resolved** if:

1. **Position is null**: `position === null` in the API response means the code has changed and the comment is outdated
2. **Author replied**: The PR author posted a reply containing resolution indicators:
   - "Fixed", "Done", "Addressed", "Resolved"
   - "Fixed in the latest push" (pr-resolver's own reply format)
3. **Bot-specific resolution**: Some bots mark their own comments as resolved when the fix is detected

**Do NOT consider a comment resolved just because:**
- Someone reacted with an emoji (reactions are not resolutions)
- A different reviewer replied (only the PR author's replies count)
- The comment is old (age is not a resolution indicator)

---

## Handling Duplicate Comments

Multiple bots may flag the same issue on the same line. When this happens:

1. Group comments by `path` + `line` (or overlapping `start_line`..`line` ranges)
2. If multiple bots flagged the same issue, pick the one with the most actionable suggestion
3. Prefer codespect's AI Fix Prompt over coderabbit's Prompt for AI Agents (codespect prompts are more structured)
4. Apply the fix once, then reply to ALL comments on that line/range referencing the same fix
5. Do NOT apply the same fix twice
