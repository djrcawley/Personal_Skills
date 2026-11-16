---
name: review-branch
description: Review a local Git branch against master or another named base. Show LGTM-style findings in Copilot Chat with colored severity markers and links to the affected lines.
---

# Review a branch

Act as a senior engineer reviewing the author's proposed branch for merge. By default, use read-only Git commands and static inspection of the patch, callers, and tests. Run tests, interpreters, or reproduction snippets only when the user explicitly requests runtime verification (for example, `/review-branch run relevant tests`). In that mode, use a focused check in the active shell and verify that it exercises the reviewed revision. If it fails to run, report that limitation separately from any findings supported by the source.

## Gather the review input

Start inside the repository the user wants reviewed. Use this sequence in PowerShell. For another shell, adapt the variable syntax and keep the same Git arguments. Replace `master` and `HEAD` only when the user specifies other refs.

```powershell
$repoRoot = git rev-parse --show-toplevel
if ($LASTEXITCODE -ne 0) { throw 'Could not locate the repository.' }
$base = git -C $repoRoot rev-parse --verify --end-of-options 'master^{commit}'
if ($LASTEXITCODE -ne 0) { throw 'Could not resolve the base.' }
$tip = git -C $repoRoot rev-parse --verify --end-of-options 'HEAD^{commit}'
if ($LASTEXITCODE -ne 0) { throw 'Could not resolve the tip.' }
$ancestor = git -C $repoRoot merge-base $base $tip
if ($LASTEXITCODE -ne 0) { throw 'Could not find the merge base.' }
git -C $repoRoot diff --name-status --find-renames $ancestor $tip --
git -C $repoRoot --no-pager diff --no-color --no-ext-diff --no-textconv --find-renames $ancestor $tip --
```

Carry the resolved root and IDs into later tool calls. If the patch is too large or output is truncated, use the same diff command with batches of paths after `--`, including both paths of a rename. Account for every path in the inventory before reporting a clean review. Report any Git failure or gap in coverage.

Use the user's change description and review criteria. For additional intent, read `git -C $repoRoot log --format=%B "$base..$tip"`. Check for root-level conventions with `git -C $repoRoot ls-tree --name-only $ancestor -- AGENTS.md`. If present, read `git -C $repoRoot show "${ancestor}:AGENTS.md"`. Review edits, additions, and deletions of that file as proposed changes.

## Review the patch

Examine every changed file. Trace interactions across functions, classes, and files. Look for bugs, security issues, discrepancies with the stated intent, confusing code, useful refactorings, and typos. Include minor findings and plausible issues that depend on a stated assumption. Ground each comment in the patch or supporting context.

Start with the patch. Before reporting a regression, trace a concrete input or caller at the common ancestor and tip. When a refactor moves setup into a helper, class, or factory, follow the default path and any supported caller override that could bypass that setup. Identify which defaults, validation, or callbacks each path receives before and after the change. Report what the branch introduced, worsened, or made reachable. Apply the usual rubric to readability, refactoring, and typo findings.

Read more code only to answer a specific open question. Use `git -C $repoRoot grep -n -F -e '<symbol>' $tip -- '<path>'` to locate a caller or test, and `git -C $repoRoot show "${tip}:<path>"` for committed source. Substitute the ancestor and old path when checking earlier behavior. Stop expanding that investigation when the evidence settles it.

Anchor each comment to the first affected added line in the reviewed revision. Check the exact statement against the tip before linking it. The diff hunk or `git grep -n` at the tip can provide that check. In a unified diff hunk, the number after `+` in the `@@` header starts the new-file line counter. Context and added lines advance it. Removed lines do not. Flag a removal when losing that behavior creates a problem. Anchor it to the next surviving line, or use `0` if none follows. Use the file's destination path for a rename and its former path for a deleted file.

Explain the specific problem and suggest a correction when helpful. Omit praise, change summaries, formatting-only remarks, and diagnostics that the compiler or type checker alone would produce. Include behavioral bugs even when a test could detect them.

## Score each finding

Score by the impact **if the finding is real**. When relevant evidence is missing, state the assumption in the comment. Uncertainty can reduce the score by at most one level, with a floor of 1.

| Chat label | Meaning |
| --- | --- |
| 🔴 5/5 · Critical | Critical impact, such as a security flaw, data loss, or a crash in a common workflow. |
| 🟠 4/5 · Bug | A defect that makes the program behave incorrectly. |
| 🟡 3/5 · Concern | A probable edge-case bug or a substantial maintenance problem. |
| 🔵 2/5 · Improvement | A small improvement to readability or structure. |
| ⚪ 1/5 · Nitpick | A nitpick worth mentioning. |

## Display the review in Copilot Chat

Write the review directly in Markdown. Show severity 2 and above by default. Honor a requested threshold, including 1 to show nitpicks. Group visible findings by file. Order the files by their highest severity and the findings within each file by descending severity.

Use a heading with the file path for each group. Start each finding with its bold chat label from the table and a clickable location. Put the explanation directly beneath it. Use the following layout with the real paths, line numbers, and findings from this review. Render the Markdown directly in chat.

### [src/C#/Connection Pool.cs](src/C%23/Connection%20Pool.cs)

- **🟠 4/5 · Bug** · [Line 48](src/C%23/Connection%20Pool.cs#L48)

  > An empty batch returns before the connection is released. Repeated empty batches can exhaust the pool. Move the release into a `finally` block.

Build links from the actual path relative to the open workspace folder. Percent-encode each raw path segment once (including spaces, literal `#`, and literal `%`), join segments with `/`, then append the one-based `#L<number>` fragment. Keep the displayed filename readable. For a file-level finding, use a file link without a fragment. If edits or a different checkout moved the line, link to the matching workspace line only if verified. Otherwise link to the file and state the reviewed ref and line in text. For a deleted file or a path outside the open workspace folders, show the repository-relative path and revision location as plain text.

For an empty diff say “No changes to review.” If a completed review found no issues, say “No findings.” If every finding fell below the display threshold, give the threshold and the number hidden so the user can choose to show them.
