---
name: review-branch
description: Review a local Git branch against master or another named base. Show LGTM-style findings in Copilot Chat with colored severity markers and links to affected lines.
---

# Review a branch

Review the proposed branch as a senior engineer. By default, use read-only Git commands and static inspection. Run tests, interpreters, or reproduction code only when explicitly requested (for example, “run relevant tests”). Verify runtime checks exercise the reviewed revision; report execution failures separately from source-supported findings.

## Resolve the review range

Start in the target repository. Use this PowerShell sequence, adapting syntax for another shell. Replace master or HEAD only when the user specifies different refs.

```powershell
$repoRoot = git rev-parse --show-toplevel
if ($LASTEXITCODE -ne 0) { throw 'Could not locate the repository.' }
$base = git -C $repoRoot rev-parse --verify --end-of-options 'master^{commit}'
if ($LASTEXITCODE -ne 0) { throw 'Could not resolve the base.' }
$tip = git -C $repoRoot rev-parse --verify --end-of-options 'HEAD^{commit}'
if ($LASTEXITCODE -ne 0) { throw 'Could not resolve the tip.' }
$ancestor = git -C $repoRoot merge-base $base $tip
if ($LASTEXITCODE -ne 0) { throw 'Could not find the merge base.' }
git -C $repoRoot status --short
git -C $repoRoot diff --name-status --find-renames $ancestor $tip --
git -C $repoRoot --no-pager diff --no-color --no-ext-diff --no-textconv --find-renames $ancestor $tip --
```

This reviews committed changes from the merge base through tip. If the worktree is dirty, say its uncommitted changes are excluded unless the user asks to include them. Carry the resolved root and IDs into later commands. If output is too large, diff batches of inventoried paths, including both sides of renames. Account for every path and report Git failures or coverage gaps.

Use the user's change description and criteria. Read commit messages when useful for intent. Read every AGENTS.md governing changed paths at the ancestor, including nested instructions; if one changes, consider both its old instructions and proposed edit.

## Review the patch

Inspect every changed file. Focus on actionable bugs, security issues, and discrepancies with intent; include useful smaller concerns, with assumptions stated. Before reporting a regression, trace a concrete input or caller at the common ancestor and tip. When setup moves into a helper, class, or factory, check the default route and supported overrides; compare the defaults, validation, and callbacks each receives.

Read more source only to resolve a specific question. Locate callers or tests with git grep and read committed files with git show at tip; use the ancestor and old path for earlier behavior. If an IDE read fails on a path, fall back to git show instead of retrying path variants. Stop when evidence settles the question. Report behavior the branch introduced, worsened, or made reachable. Omit praise, formatting-only remarks, and diagnostics a compiler or type checker alone would catch.

Anchor each finding to the first affected added line and verify it at tip. In a unified hunk, the new-file counter starts at the number after + and advances for context and added lines, not removals. For a removal, use the next surviving line or 0 if none follows. Use the destination path for a rename and former path for a deleted file.

## Score findings

Score impact if the finding is real. State material assumptions. Uncertainty may lower the score by at most one level, with a floor of 1.

| Chat label | Meaning |
| --- | --- |
| 🔴 5/5 · Critical | Security flaw, data loss, or common-workflow failure. |
| 🟠 4/5 · Bug | Incorrect behavior. |
| 🟡 3/5 · Concern | Probable edge-case bug or substantial maintenance problem. |
| 🔵 2/5 · Improvement | Small, meaningful readability or structure improvement. |
| ⚪ 1/5 · Nitpick | Minor issue. |

## Display in Copilot Chat

Write Markdown directly in chat. Show severity 2+ by default; honor a requested threshold. Group by file, ordering files by highest severity and findings within each file by descending severity. Use this structure:

### [relative/path](encoded/relative/path)

- **🟠 4/5 · Bug** · [Line 48](encoded/relative/path#L48)

  > State the specific problem and suggest a correction when helpful.

Build links from the actual path relative to the open workspace. Percent-encode each raw path segment once (including spaces, #, and %), join with /, then append the one-based #L line fragment. Verify the linked line at tip. For file-level findings, omit the fragment. If the line cannot be verified in the open workspace, or the file is deleted or outside it, show the repository path and reviewed revision location as plain text.

For an empty diff say “No changes to review.” If review is complete with no issues, say “No findings.” If all findings fall below the threshold, state the threshold and count hidden.
