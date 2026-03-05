---
name: Code Time Traveler
description: Analyzes git history to identify when bugs were introduced and suggests optimal fix points
version: 1.0.0
author: OpenClaw Team
tags:
  - git
  - debugging
  - history-analysis
  - bug-tracking
  - blame
  - bisect
dependencies:
  - git >= 2.30
  - bash >= 4.0
  - jq (optional, for JSON formatting)
  - tig (optional, for interactive view)
environment:
  GIT_DIR: ".git"
  GIT_PAGER: "cat"
  CTT_LOG_LEVEL: "INFO"
  CTT_MAX_COMMITS: "1000"
  CTT_BISECT_AUTO: "false"
---

# Code Time Traveler

A specialized skill for analyzing git commit history to pinpoint when bugs were introduced, track code evolution, and determine the optimal points for fixes.

## Purpose

Real-world use cases:
- "The API started returning 500 errors after deployment on March 3rd - which commit caused this?"
- "When was the `processPayment()` function changed last and who modified it?"
- "Find the commit that introduced the memory leak we're seeing in production"
- "Show me the history of changes to `src/auth/middleware.ts` with context about each change"
- "We have a regression bug - use bisect to find the bad commit automatically"
- "Which commits modified the database schema between v2.1.0 and v2.3.0?"
- "Trace the evolution of this specific line of code across all branches"
- "Identify all commits that touched the `payment_processing` module in the last quarter"

## Scope

### Commands

**`analyze-bug [file] [line] [error-message]`**
Analyzes git history around a specific error location to find likely culprit commits.

Parameters:
- `file`: Path to file containing the bug (required)
- `line`: Line number where error occurs (required)
- `error-message`: Exact error text or pattern (optional but recommended)

Example: `analyze-bug src/utils/api.ts 127 "Connection timeout after 30s"`

**`bisect-start [good-commit] [bad-commit] [test-command]`**
Automates git bisect to find the commit that introduced a bug.

Parameters:
- `good-commit`: Known working commit hash/tag/branch (default: HEAD~20)
- `bad-commit`: Known broken commit (default: HEAD)
- `test-command`: Shell command that returns 0 for good, non-0 for bad (required)

Example: `bisect-start v2.1.0 HEAD "npm test -- --grep 'payment flow'"`

**`blame-deep [file] [since]`**
Enhanced git blame with commit context, showing commit messages, authors, and adjacent changes.

Parameters:
- `file`: File to analyze (required)
- `since`: Filter commits from date (e.g., "2 weeks ago", "2025-01-01") (optional)

Example: `blame-deep src/models/User.ts "1 month ago"`

**`timeline [path] [days]`**
Generates chronological timeline of changes to a path with statistical summary.

Parameters:
- `path`: File or directory path (default: current directory)
- `days`: Number of days to analyze (default: 30)

Example: `timeline src/components/ 90`

**`find-introduction [pattern] [files]`**
Locates when a specific code pattern, function, or import was first added.

Parameters:
- `pattern`: Regex pattern to search for (required)
- `files`: File glob pattern (default: "**/*.{js,ts,jsx,tsx}")

Example: `find-introduction "useReducer" "src/hooks/**/*.ts"`

**`compare-releases [tag1] [tag2] [filter]`**
Compares two releases to identify all changes in a specific component.

Parameters:
- `tag1`: First release tag (required)
- `tag2`: Second release tag (required)
- `filter`: Filter by path, author, or message (e.g., "path:src/auth/", "author:john")

Example: `compare-releases v2.3.0 v2.4.0 "path:src/payment/"`

**`hotspot-analysis [days] [extensions]`**
Identifies files with highest churn rate (most frequently changed) in given period.

Parameters:
- `days`: Analysis window (default: 30)
- `extensions`: File extensions to include (default: ".js,.ts,.tsx,.jsx,.py,.go,.rs")

Example: `hotspot-analysis 60 ".js,.ts,.tsx"`

**`regression-search [date] [keywords]`**
Search for commits on or after a date that may have introduced regressions.

Parameters:
- `date`: Date to start search (default: last successful deployment date from .openclaw/deploy.log)
- `keywords`: Keywords suggesting risky changes: "refactor", "perf", "security", "fix", "breaking"

Example: `regression-search "2025-03-01" "refactor,perf"`

## Work Process

### Analyzing a Bug Introduction

1. User provides file, line number, and error message
2. Skill runs `git blame` on the specific line to identify last modification commit
3. Executes `git show <commit>` to examine changes in that commit
4. Checks commit message for keywords like "fix", "refactor", "breaking" that suggest intentional changes
5. Looks at adjacent commits (±5) to understand context and whether bug was introduced as side effect
6. Queries `git log -p --follow -L <line>:<file>` to see full evolution of that line across renames
7. Outputs:
   - Commit hash, author, date
   - Full commit diff
   - Related commits that touched same area
   - Suggested fix point (revert, patch, or cherry-pick from earlier version)
   - Risk assessment: "High confidence: direct change to buggy line" vs "Medium: adjacent changes"

### Automated Bisection

1. User provides test command that reproduces bug
2. Skill initializes git bisect: `git bisect start`
3. Marks current HEAD as bad: `git bisect bad`
4. Marks provided good commit: `git bisect good <good-commit>`
5. Enters bisect loop automatically (unless CTT_BISECT_AUTO=false):
   - For each commit, runs test command
   - Logs result with commit info
   - Calls `git bisect good` or `git bisect bad`
6. When bisect converges, outputs:
   - First bad commit details
   - Full diff of that commit
   - Commits between good and bad for context
   - Recommendation: "Revert this commit" or "Apply minimal fix on top"
7. Resets bisect: `git bisect reset`

### Hotspot Identification

1. Skill runs: `git log --since="<days> days ago" --oneline --name-only | sort | uniq -c | sort -nr`
2. Parses output to rank files by number of commits touching them
3. For top 10 hotspots, calculates:
   - Number of distinct authors
   - Ratio of bug-fix commits vs feature commits (using message analysis)
   - Average time between commits (churn frequency)
4. Outputs ranked list with metrics and risk score (high churn + many authors = higher risk)

## Golden Rules

1. **Never suggest a fix without verifying the commit exists and is reachable** - Always run `git cat-file -e <commit>` before recommending actions
2. **Consider merge commits separately** - Use `git log --no-merges` for initial analysis, then explicitly check merges if needed
3. **Account for file renames** - Use `git log --follow` when tracking file history across renames
4. **Bisect requires clean working tree** - Verify `git status --porcelain` returns empty before starting
5. **Check commit signing** - Prefer verified commits: `git log --show-signature -1 <commit>`
6. **Context is critical** - Always show at least 3 commits before and after the suspect commit
7. **Performance matters** - Limit log searches with `--since` and file globs; use `git rev-list --count` instead of counting full logs
8. **Branches must be specified explicitly** - When comparing releases, verify tags exist: `git tag -l | grep -E "^<tag>$"`
9. **Never modify history** - This skill only analyzes; never amend, rebase, or force-push
10. **Document assumptions** - If using heuristics (e.g., "fix" keyword), state confidence level

## Examples

### Example 1: Find bug introduction in a specific file

**User input:**
```
analyze-bug src/server/request.js 243 "Cannot read property 'headers' of undefined"
```

**Skill output:**
```
Analyzing bug location: src/server/request.js:243
==============================================

1. BLAME ANALYSIS (last modification of line 243):
   Commit:  7a3f9c2d
   Author:  jane.doe@company.com
   Date:    2025-02-28 14:32:18 -0500
   Message: feat: add request validation middleware

2. COMMIT DIFF:
   commit 7a3f9c2d8f4e2a1b...
   Author: Jane Doe <jane.doe@company.com>
   Date:   Fri Feb 28 14:32:18 2025 -0500

       feat: add request validation middleware

       - Add validation for incoming requests
       - Reject requests missing Content-Type header
       - Log validation failures

   diff --git a/src/server/request.js b/src/server/request.js
   index 5a8b2c1..d4e5f6a 100644
   --- a/src/server/request.js
   +++ b/src/server/request.js
   @@ -240,7 +240,12 @@ function processRequest(req, res) {
     // Validate request
   - if (!req.headers) {
   -   throw new Error('Missing headers');
   - }
   + const validationErrors = validateRequest(req);
   + if (validationErrors.length > 0) {
   +   logger.warn('Validation failed:', validationErrors);
   +   return res.status(400).json({ error: validationErrors });
   + }

   - // Process request (line 243 was shifted)
     await processHandler(req, res);

3. ADJACENT COMMITS:
   - 7a3f9c2d (above) - current
   - b8c4d9e1 - refactor: simplify error handling (2 days before)
   - 3f2a1b4c - fix: handle null headers gracefully (5 days before)

4. ANALYSIS:
   The line 243 ("await processHandler(req, res);") existed before commit 7a3f9c2d.
   That commit added validation logic ABOVE line 243, but did NOT modify line 243 itself.
   However, the validation now returns early for invalid requests, meaning line 243
   is only executed when validation passes. The error "Cannot read property 'headers'
   of undefined" suggests that `req` itself is undefined, not just `req.headers`.

   RECOMMENDATION: Medium confidence that commit 7a3f9c2d introduced the bug as a
   side effect. The validation assumes `req` exists, but upstream code may now
   call processRequest() with undefined in some edge cases. Check commit 3f2a1b4c
   which explicitly handled null headers - this may have been removed in 7a3f9c2d.

5. SUGGESTED FIX POINT:
   - Option A (revert validation): Revert portions of 7a3f9c2d to restore null check
   - Option B (defensive coding): Add `if (!req) throw new Error('Request is undefined')`
     at the start of processRequest()
   - Option C (bisect): Run automated bisect to confirm if 7a3f9c2d is truly first bad

Confidence: 65% | Evidence: Line 243 unchanged, but surrounding logic altered
```

### Example 2: Automated bisect to find regression

**User input:**
```
bisect-start v2.5.0 HEAD "curl -f http://localhost:3000/api/health || exit 1"
```

**Skill output:**
```
Starting bisect to find regression between v2.5.0 (good) and HEAD (bad)
===============================================================

Verifying repository state:
  Working tree: clean
  Current branch: feature/auth-overhaul
  HEAD: abc1234de

Initializing git bisect...
  git bisect start
  git bisect bad  # HEAD is bad
  git bisect good v2.5.0  # Tag v2.5.0 is good

Testing commit [10/156]: d9e8f7a6 (3 days ago)
  Command: curl -f http://localhost:3000/api/health || exit 1
  Result: SUCCESS (0)
  Marking as: GOOD

Testing commit [9/156]: b4c3d2e1 (4 days ago)
  Command: curl -f http://localhost:3000/api/health || exit 1
  Result: FAILURE (22)
  Error: curl: (22) The requested URL returned error: 500
  Marking as: BAD

Testing commit [8/155]: a1b2c3d4 (5 days ago)
  Command: curl -f http://localhost:3000/api/health || exit 1
  Result: SUCCESS (0)
  Marking as: GOOD

[... skipping intermediate commits for brevity ...]

Testing commit [1/1]: f1e2d3c4 (11 days ago)
  Command: curl -f http://localhost:3000/api/health || exit 1
  Result: SUCCESS (0)
  Marking as: GOOD

BISECT COMPLETE
===============

First bad commit: b4c3d2e1
  Hash:        b4c3d2e1a5f6e7d8c9b0a1
  Author:      mike.smith@company.com
  Date:        2025-02-24 09:15:33 -0500
  Message:     fix: cache database connections

Full diff:
  commit b4c3d2e1a5f6e7d8c9b0a1
  Author: Mike Smith <mike.smith@company.com>
  Date:   Mon Feb 24 09:15:33 2025 -0500

      fix: cache database connections

      - Add Redis-based connection pooling
      - Cache DB connections for 5 minutes
      - Reduces connection overhead

  diff --git a/src/db/connection.js b/src/db/connection.js
  index 8a9b0c1..2d3e4f5 100644
  --- a/src/db/connection.js
  +++ b/src/db/connection.js
  @@ -45,6 +45,12 @@ function getConnection() {
       return connectionPool.get();
     }

  +  // Cache connections in Redis
  +  const cached = await redis.get('db:connection');
  +  if (cached) {
  +    return deserialize(cached);
  +  }
  +
     const conn = await createNewConnection(config);
  +  await redis.setex('db:connection', 300, serialize(conn));
     return conn;
   }

Context: This is the only commit between v2.5.0 (good) and the bad state. The commit
introduces Redis caching for database connections. The health endpoint likely depends
on a database query that now fails when Redis is unavailable or when serialization
fails.

VERIFICATION:
  - Check Redis connectivity: `redis-cli ping`
  - Review logs: `tail -100 logs/error.log | grep -i redis`
  - Test health endpoint directly after commit: `git checkout b4c3d2e1 && npm start`

RECOMMENDATION:
  Option 1 (revert): git revert b4c3d2e1
  Option 2 (fix): Add Redis availability check and fallback to direct connection
  Option 3 (config): Ensure Redis is deployed with the new version

Run `git bisect reset` to exit bisect mode.
```

### Example 3: Deep blame with commit context

**User input:**
```
blame-deep src/auth/jwt.js "2 weeks ago"
```

**Skill output:**
```
Deep blame analysis: src/auth/jwt.js
===================================

Line-by-line with commit context:
─────────────────────────────────────────────────────────────────────────────
  1  const jwt = require('jsonwebtoken');                   │ 3f2a1b4c │ Alice │ 2025-02-10
     └─ fix: add null check for headers (5 days ago)
  2  const config = require('../config');                  │ abc1234d │ Bob   │ 2025-02-15
     └─ chore: update config import path
  3                                                                             
  4  function verifyToken(token) {                         │ 7a3f9c2d │ Jane  │ 2025-02-28
     └─ feat: add request validation middleware (3 weeks ago)
  5    if (!token) {                                       │ 7a3f9c2d │ Jane  │ 2025-02-28
  6      throw new Error('Token required');                │ 7a3f9c2d │ Jane  │ 2025-02-28
  7    }                                                   │ 7a3f9c2d │ Jane  │ 2025-02-28
  8    return jwt.verify(token, config.secret);           │ 3f2a1b4c │ Alice │ 2025-02-10
     └─ fix: add null check for headers (5 days ago)
  9  }                                                     │ 3f2a1b4c │ Alice │ 2025-02-10
 10                                                                            
 11  function generateToken(user) {                        │ def56789 │ Carol │ 2025-01-20
     └─ feat: implement token generation (6 weeks ago)
 12    const payload = {                                   │ def56789 │ Carol │ 2025-01-20
 13      userId: user.id,                                  │ def56789 │ Carol │ 2025-01-20
 14      role: user.role,                                  │ def56789 │ Carol │ 2025-01-20
 15      iat: Math.floor(Date.now() / 1000)               │ def56789 │ Carol │ 2025-01-20
 16    };                                                  │ def56789 │ Carol │ 2025-01-20
 17    return jwt.sign(payload, config.secret);           │ def56789 │ Carol │ 2025-01-20
 18  }                                                     │ def56789 │ Carol │ 2025-01-20

SUMMARY:
  Total commits touching this file (last 2 weeks): 4
  Lines unchanged since 2025-02-10: 8 (lines 1, 5-7, 9, 12-16, 18)
  Most recent change: 7a3f9c2d by Jane Doe on 2025-02-28
  Authors: Alice (2 commits), Jane (1), Bob (1), Carol (1)

RISK INDICATORS:
  - Line 8 (jwt.verify call) was last modified 5 days ago in a "fix" commit
  - Lines 1-2 (imports) have 3 different authors in 2 weeks - high churn
  - No changes in last 3 days - stable period

RECOMMENDATION:
  If you're debugging JWT issues, start with commit 3f2a1b4c (last fix to verifyToken)
  and review 7a3f9c2d (most recent change to the file structure).
```

## Rollback Commands

**Revert a single commit:**
```bash
git revert <commit-hash>
# Creates a new commit that undoes the changes
# Safe for shared branches
```

**Reset to a previous state (DESTRUCTIVE - local only):**
```bash
git reset --hard <commit-hash>
# WARNING: discards all commits after <commit-hash>
# Use only on unshared branches
```

**Cherry-pick a fix from another branch:**
```bash
git checkout feature/fix-branch
git cherry-pick <good-commit-hash>
```

**Interactive rebase to edit/squash commits:**
```bash
git rebase -i <commit-hash>^
# Mark commit as "edit" to modify, or "squash" to merge
# Only use on private/unpushed commits
```

**Manual revert of specific lines:**
```bash
# Checkout specific file version from earlier commit
git checkout <good-commit-hash> -- path/to/file.js
# Then commit the restored version
git commit -m "revert: restore working version of file.js from <good-commit>"
```

**Bisect reset (cleanup):**
```bash
git bisect reset
# Returns branch to original HEAD and disables bisect
```

**Partial revert (only specific hunks):**
```bash
git revert -n <commit-hash>
# Stage only selected changes with git add -p
git commit -m "revert: partial revert of <commit-hash>"
```

## Verification Steps

After identifying a suspect commit:
1. **Checkout test**: `git checkout <commit-hash>` and run your test suite to confirm bug appears
2. **Range check**: `git log --oneline <good-commit>..<bad-commit>` to see all commits in range
3. **Diff review**: `git diff <good-commit> <bad-commit> -- path/to/file` to isolate changes
4. **Blame cross-check**: `git blame <bad-commit> -- path/to/file` to see line ownership in broken state
5. **Build verification**: `git checkout <commit-hash> && npm test` (or equivalent) to reproduce

**Successful analysis requires:**
- Git repository with sufficient history (minimum 20 commits recommended)
- Linear history (merge commits can complicate bisect; use `--first-parent` if needed)
- Test command that reliably returns 0 for good, non-0 for bad
- Clean working tree before bisect operations

## Troubleshooting

**"fatal: ambiguous argument 'HEAD' - unknown revision or path not in the working tree"**
- Ensure you're inside a git repository: `git rev-parse --is-inside-work-tree`
- Check that the branch/tag exists: `git tag -l` or `git branch -a`

**"bisect start... you need to start by 'git bisect start'..."**
- Another bisect session is already active. Run `git bisect reset` first.

**"some test runs with no changes" during bisect**
- Your test command is not deterministic. Add explicit seed: `npm test -- --seed=12345`
- Or use `CTT_BISECT_AUTO=false` to manually verify each commit

**Too many commits to analyze (performance)**
- Set `CTT_MAX_COMMITS` environment variable to limit analysis scope
- Add `--since` flags: `timeline --since="1 month ago"`
- Use specific file globs: `hotspot-analysis 30 "src/components/*.tsx"`

**Blame shows "Not Committed Yet"**
- File has uncommitted changes. Run `git status` and commit or stash changes first.

**Cannot find introduction of pattern**
- Pattern may have been renamed or moved. Use `git log --all -S'<pattern>'` to search content changes
- Or use `find-introduction` with broader pattern or check across branches: `git log --all --oneline --grep='<pattern>'`

**Conflicting bisect results**
- Test command is flaky. Add retries: `for i in {1..3}; do <test> && break || sleep 1; done`
- Or use `CTT_BISECT_AUTO=false` to manually verify each step

**"error: pathspec '<file>' did not match any files"**
- File was deleted in current branch. Use `--follow` with explicit commit: `git log --follow -- <file>`
- Or check if file exists in another branch: `git checkout <branch> -- <file>`
```