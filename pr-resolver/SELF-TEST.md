# PR Resolver — Self-Test

## Trigger Phrases

The following phrases should activate the pr-resolver skill:

| Phrase | Expected Behavior |
|--------|-------------------|
| "resolve pr comments" | Detect current PR, start fix loop |
| "fix pr review" | Same as above |
| "fix review comments" | Same as above |
| "address pr feedback" | Same as above |
| "pr-resolver" | Same as above |
| "resolve pr #42" | Use PR #42 explicitly |
| "fix review for https://github.com/org/repo/pull/42" | Parse PR from URL |
| "fix all pr comments automatically" | Enable auto mode, skip user prompts for major |
| "resolve pr comments --auto" | Enable auto mode |
| "address review from codespect" | Start fix loop, expect codespect-enhanced handling |

## Functional Tests

### Test 1: Codespect Comment Parsing

Given this codespect comment body:

````markdown
**Avoid N+1 queries**

The command issues a separate COUNT() query for every daily metric.

<details>
<summary>Code Suggestion</summary>

```suggestion
$counts = Model::query()->groupBy('campaign', 'date')->get();
```
</details>

<details>
<summary>AI Fix Prompt</summary>

> Copy-paste this prompt into your AI assistant to get a fix for this issue.

```text
I have a code review comment on my pull request that I need to address.

File: `app/Console/Commands/SyncClicks.php` (line 44)

Reviewer's comment:
**Avoid N+1 queries**

The command issues a separate COUNT() query for every daily metric.

Suggested fix:
$counts = Model::query()->groupBy('campaign', 'date')->get();

Please update the code in this file to address the reviewer's feedback.
```

</details>
````

**Expected extractions:**
- Title: "Avoid N+1 queries"
- Severity: minor (no "Nitpick:" prefix)
- AI Fix Prompt: The full text block starting with "I have a code review comment..."
- Code Suggestion: `$counts = Model::query()->groupBy('campaign', 'date')->get();`
- File: `app/Console/Commands/SyncClicks.php`
- Line: 44

### Test 2: Codespect Nitpick Detection

Given title: `**Nitpick: redundant index**`

**Expected:** Severity = nitpick, auto-fixable = yes

### Test 3: CodeRabbit Severity Parsing

Given this coderabbit comment opening:

```markdown
_Potential issue_ | _Minor_
```

**Expected:** Category = "Potential issue", Severity = minor, auto-fixable = yes

Given:

```markdown
_Potential issue_ | _Critical_
```

**Expected:** Category = "Potential issue", Severity = critical, auto-fixable = no (always ask user)

### Test 4: Duplicate Comment Detection

Given two comments on the same file and line:
- Comment A from `codespect-io[bot]` on `app/Controller.php:34`
- Comment B from `coderabbitai[bot]` on `app/Controller.php:34`

Both flag the same URL construction issue.

**Expected behavior:**
- Detect overlap by matching `path` + `line`
- Apply fix once (prefer codespect's AI Fix Prompt)
- Reply to both comments referencing the same fix

### Test 5: Unresolved Comment Filtering

Given these comments:
- Comment with `position: null` → SKIP (outdated)
- Comment with a reply from PR author saying "Fixed" → SKIP (resolved)
- Comment with no replies → INCLUDE (unresolved)
- Comment from PR author themselves → SKIP (self-comment)
- Top-level review body with no path/line → SKIP (summary, not actionable)

### Test 6: Bottom-to-Top Processing

Given comments on `app/Service.php` at lines 10, 50, and 100:

**Expected processing order:** line 100 first, line 50 second, line 10 last.

### Test 7: Iteration Termination

- After push, no new unresolved comments → terminate with success
- After 5 iterations with remaining comments → terminate with report
- Same comments persist for 2 consecutive iterations → pause and ask user

### Test 8: Auto Mode vs Interactive Mode

Given a major-severity comment:
- Interactive mode (default) → ask user for approach
- Auto mode → apply best AI judgment without asking
- Critical severity → ask user regardless of mode

## Integration Test

To validate the full workflow end-to-end:

1. Create a test branch with intentional issues
2. Open a PR
3. Wait for codespect-io to review
4. Run pr-resolver
5. Verify:
   - All comments were fetched
   - Fixes were applied correctly
   - Replies were posted on GitHub
   - Commit message follows the expected format
   - Second iteration finds no new comments
   - Final report is accurate
