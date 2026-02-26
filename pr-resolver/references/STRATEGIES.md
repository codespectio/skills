# Fix Strategies & Patterns

Reference guide for how to approach, batch, commit, and verify review comment fixes.

## Severity Classification Decision Tree

```
Is the commenter a bot?
├── Yes → Check bot type
│   ├── codespect-io → Check title prefix
│   │   ├── "Nitpick:" prefix → NITPICK (auto-fix)
│   │   ├── Review summary says "minor" → MINOR (auto-fix)
│   │   └── Body mentions security/architecture/breaking → MAJOR (ask user)
│   ├── coderabbitai → Check severity label
│   │   ├── 🟢 Info → SKIP (no fix needed)
│   │   ├── 🟡 Minor → MINOR (auto-fix)
│   │   ├── 🔴 Critical → CRITICAL (always ask user)
│   │   └── 🧹 Nitpick → NITPICK (auto-fix)
│   └── Other bot → AI judgment → MINOR or MAJOR
└── No (human reviewer) → Analyze comment scope
    ├── Style/naming/formatting → NITPICK (auto-fix)
    ├── Small logic fix, add guard, remove unused code → MINOR (auto-fix)
    ├── Refactor involving 1-2 files, <30 lines → MINOR (auto-fix)
    ├── Architecture change, API redesign, data model change → MAJOR (ask user)
    ├── Security-critical fix → CRITICAL (always ask user)
    └── Ambiguous → default to MAJOR (ask user)
```

## Fix Application Order

### Within a Single File

Always process comments **bottom-to-top** (highest line number first):

```
File: app/Services/PaymentService.php
  Comment on line 142 → fix THIRD
  Comment on line 87  → fix SECOND
  Comment on line 23  → fix FIRST (processed last, applied to lines not shifted)
```

Wait — this reads confusingly. To clarify: we **process** in reverse line order, meaning we **apply** line 142 first, then 87, then 23. This way, fixing line 142 doesn't shift the line numbers for the comments at 87 and 23.

### Across Files

Process files in alphabetical order for predictable behavior:

```
1. app/Http/Controllers/PaymentController.php
2. app/Models/Payment.php
3. tests/Feature/PaymentTest.php
```

### Overlapping Comments

When multiple comments reference overlapping line ranges in the same file:

1. Identify the union of all overlapping ranges
2. Read the entire range plus 10 lines of context on each side
3. Apply all fixes for that range in a single edit operation
4. Reply to all comments in the overlap, referencing the combined fix

## Commit Strategies

### Single-Concern Commit (Default)

When all review comments are about similar concerns (e.g., all performance fixes, or all style fixes):

```
fix(review): address PR review comments

- Avoid N+1 query in SyncColdEmailClickCounts command
- Add guard clause to ColdEmailDailyMetric saving hook
- Remove redundant created_at index from clicks migration
- Preserve query params in redirect URL construction

Resolves 4 review comments from codespect-io
```

### Multi-Concern Commits

When fixes span different concerns, split into logical commits:

```
# Commit 1: Performance fixes
fix(perf): resolve N+1 queries flagged in review

- Aggregate click counts in single query instead of per-metric loop
- Guard saving hook to avoid unnecessary COUNT queries

# Commit 2: Bug fixes
fix(url): preserve existing query parameters in redirect

- Check for existing '?' before appending UTM params

# Commit 3: Cleanup
chore: remove redundant database index

- Drop standalone created_at index covered by composite index
```

### When to Split

Split commits when:
- Fixes address fundamentally different concerns (perf vs bug vs style)
- A single fix is large enough to warrant its own commit (>30 lines changed)
- Fixes touch different parts of the codebase with no overlap

Keep as single commit when:
- All fixes are minor/nitpick
- Fixes are all in the same file or closely related files
- Total changes are small (<50 lines across all fixes)

## Handling Conflicting Suggestions

When multiple reviewers (or bots) suggest different fixes for the same issue:

### Same Issue, Different Approaches

1. Compare the suggested approaches
2. Evaluate which is more idiomatic for the codebase
3. Check if either approach has side effects the other doesn't
4. Prefer the more conservative fix (less risk of regression)
5. In the reply, acknowledge both suggestions and explain why you chose one

Example reply:
> Fixed in the latest push. Both codespect-io and coderabbitai flagged this URL construction issue. Applied the `str_contains` check approach as it's simpler and handles the edge case without needing full URL parsing.

### Contradicting Suggestions

If two reviewers suggest genuinely contradictory changes:

1. Do NOT auto-fix
2. Escalate to the user regardless of severity
3. Present both suggestions with pros/cons
4. Let the user decide

## Verification Checklist

Before committing fixes, verify:

- [ ] Each edited file still has valid syntax (no unclosed brackets, quotes, etc.)
- [ ] No unintended whitespace changes outside the fix regions
- [ ] Import statements are consistent (if you added a new import, it follows the file's existing style)
- [ ] No commented-out code was accidentally introduced
- [ ] The diff only contains changes related to the review comments

### Optional: Run Tests

If the project has a test suite and the user didn't explicitly skip testing:

```bash
# Detect test framework
if [ -f "phpunit.xml" ] || [ -f "phpunit.xml.dist" ]; then
    php artisan test --parallel
elif [ -f "jest.config.js" ] || [ -f "jest.config.ts" ]; then
    npx jest --bail
elif [ -f "pytest.ini" ] || [ -f "pyproject.toml" ]; then
    pytest -x
fi
```

Only run tests if:
- The fixes touched logic (not just style/naming)
- Test files exist for the modified code
- The user didn't say "skip tests" or "just push"

## Reply Templates

### Standard Fix Reply

```
Fixed in the latest push. <1-2 sentence explanation of what was changed and why>.
```

### Fix With Deviation From Suggestion

```
Fixed in the latest push. Applied a slightly different approach than suggested: <explanation>. The suggestion's approach of <X> would <reason it wasn't ideal>, so instead <what was done>.
```

### Skipped Comment Reply (do NOT post this — just track internally)

Do not reply to comments you skip. Only reply when you've actually pushed a fix. Track skipped comments internally for the final report.

### Duplicate Fix Reply

When the same fix addresses multiple comments:

```
Fixed in the latest push (same fix as the one addressing <other reviewer>'s comment on this line). <brief explanation>.
```

## Iteration Behavior

### What to Expect Per Iteration

| Iteration | Typical Outcome |
|-----------|----------------|
| 1 | Fix all initial comments from first review round |
| 2 | Fix new comments that bots posted after reviewing the fixes |
| 3 | Usually clean — bots found no new issues |
| 4+ | Rare — indicates the fixes are introducing new issues |

### When Fixes Create New Issues

If iteration 2 introduces comments that weren't in iteration 1, this means the fixes triggered new findings. This is normal for the first re-review. If it keeps happening:

1. After iteration 3, pause and assess
2. Check if the new comments are on the same lines you just fixed
3. If yes → the fix approach may be wrong. Ask user for guidance.
4. If no → the new comments are on different code. Continue fixing.

### Stale Comment Detection

Between iterations, some comments may become stale (the code they reference has changed). The GitHub API indicates this with `position: null`. Always re-check this field after each push — previously active comments may become stale after your fixes.

## Edge Cases

### Comment on Deleted File
If a comment references a file that was deleted in the PR, skip it. The deletion itself may be the fix.

### Comment on Binary File
Skip comments on binary files (images, compiled assets). Report to user.

### Comment With No Code Reference
Some review comments are general observations with no specific file/line. These appear as top-level review body text or issue comments. Do not treat these as actionable fix items — they're informational.

### Comment on Test Files
Treat test file comments the same as source file comments. Do not skip them. Tests are code too.

### Comment Asking for New Tests
If a reviewer asks "please add tests for this", this is a major comment. Always ask the user, as writing new tests requires understanding the expected behavior and test framework conventions.
