---
name: ship-this
description: >
  Load this skill before shipping a UI change, to statically review the diff for design-craft regressions (accessibility, motion, responsive, and visual or UX drift), correctness and security issues, exposed secrets or credentials, and leftover commented-out code, then write a commit message and pull request description that match the repository's conventions, and offer to commit and push. Triggers on "ship this", "review this", "ready to ship". This is the ship gate at the end of the design → build → ship pipeline.
---

# Ship this

The final review before a UI change ships. It assumes the code compiles and passes lint, so it skips the mechanical style a linter enforces, but it still reviews logic, correctness, and security. The skill checks the diff for the design-craft regressions linters miss, then turns that diff into a clean commit message and PR description.

## Core mandate

Review the changed code and the minimum surrounding context needed to validate it. Do not audit unrelated parts of the codebase. Catch the regressions a linter cannot detect, such as accessibility, motion, responsive, and visual or UX drift. Then write a commit message and PR description that read as a developer wrote them, with no tooling attribution.

Every result you report comes from a command that ran or a source that was read. Never claim a secret scan passed without running a scanner, never report a commit or push that git did not confirm, and never invent findings to lengthen the review. When you can only partially verify something, say so rather than implying a clean result.

## Severity scale

| Level | Meaning |
| --- | --- |
| **Blocking** | A real bug, a leaked secret, or a shipped accessibility failure. Do not merge. |
| **Note** | Worth checking, a likely regression or a quality gap, not always wrong. |

A clean diff should pass with an empty list. Do not invent minor findings to lengthen the list, and skip formatting and naming preferences.

## The checklist

### 1. Gather the diff

Gather every pending change, so a clean result is trustworthy. Read the git state once up front, the current branch, its upstream, the default branch, and `git status --short`, so later steps reuse it instead of shelling out again. Pick the scope that matches the request.

- Uncommitted work about to be committed. Run `git diff HEAD` to capture staged and unstaged edits to tracked files.
- A whole branch headed for a PR. Run `git diff <base>...HEAD` for committed work, then add `git diff HEAD` for any uncommitted work on top.

`git diff` omits untracked files, so detect them with `git status --short` and inspect each new UI file directly. Set `<base>` to the branch the PR merges into, usually the default branch from `git symbolic-ref --short refs/remotes/origin/HEAD`, falling back to `origin/main` or `origin/master`, and to local `main` or `master` only when no remote exists. Do not use the branch's own upstream, which tracks its remote copy and would hide most of the change set. If you cannot determine the base confidently, ask before continuing.

### 2. Review the UI diff

Check the diff against each concern below. This is a static review of the code, so treat any check that needs a rendered page, a keyboard pass, or a reduced-motion run as unverified, and record it under Testing as pending.

- [ ] **Accessibility** — new interactive elements have keyboard handlers or native semantics, focus styling is present and not removed, focus order is logical, no keyboard trap, and roles and names are correct.
- [ ] **Motion** — every new animation provides an appropriate `prefers-reduced-motion` alternative, uses performant animation properties, and animates for a clear reason at a sensible frequency.
- [ ] **Responsive** — no fixed width forces overflow on a small viewport, breakpoint and container-query logic is present, and touch-target sizing and safe-area handling are correct.
- [ ] **Visual and UX drift** — design tokens replace hardcoded color, spacing, and type. The change fits the surrounding system. No magic number stands where a token exists.
- [ ] **User-facing copy** — new labels, errors, empty states, and tooltips read well. For depth, defer to the `docs-style-guide` and `writing-checklist` skills.
- [ ] **Correctness and security** — off-by-one errors, null and undefined handling, race conditions, and injection risks.
- [ ] **Secrets and credentials** — no token, key, password, or credential is introduced by the diff. See the scan below.
- [ ] **Cleanliness** — no commented-out blocks left behind. Leave `console.log` and `debugger` to the linter, which owns them.

For accessibility, motion, and responsive depth, the `keyboard-accessibility`, `motion-design`, and `responsive-design` skills in this package carry the full rules.

#### Secret and credential scan

A credential in the diff is **Blocking**. Do not merge, and tell the user to rotate it, because a committed secret is compromised even after removal.

- Prefer a real scanner over eyeballing. If `gitleaks`, `trufflehog`, `git secrets`, or `detect-secrets` is available, run it on the diff and treat its output as authoritative. Take exact match patterns from the tool's own ruleset rather than writing your own.
- When no scanner is available, fall back to a documented pattern check, and scan only added lines so a secret the diff removes does not raise a false positive. Verified GitHub token prefixes are a concrete anchor: `ghp_` classic token, `github_pat_` fine-grained token, `gho_` OAuth token, `ghu_` user-to-server token, `ghs_` installation token, `ghr_` refresh token. Also look for private-key PEM headers such as `-----BEGIN PRIVATE KEY-----`, and high-entropy values assigned to names like `password`, `secret`, `token`, or `api_key`.
- Be honest about coverage. If only the fallback check ran, report the scan as heuristic rather than implying a clean bill of health.

### 3. Write the commit message

Match the repo's existing convention. Do not assume Conventional Commits. Read `git log <base> -20 --pretty=format:"%s%n%b---"` first, then classify it.

- **Conventional Commits** (`type(scope): summary`) — follow it, and use only the types the repo already uses. Do not invent ones that never appear.
- **Ticket-prefixed** (`PROJ-123: summary` or `[PROJ-123] summary`) — use that prefix, and look for the ticket ID in the branch name if it is not obvious elsewhere.
- **Plain imperative** (`Add x`, `Fix y`) — write in that style, and do not impose a type or scope system the repo does not use.
- If the log is empty or too mixed to classify, default to Conventional Commits.

Apply these regardless of style.

- [ ] Subject line is 72 characters or fewer, in imperative mood ("add", not "added").
- [ ] Body appears only when the change needs more than the subject.
- [ ] Scope or prefix appears only when the changed paths make it obvious or the repo's pattern calls for it, and drops when the diff spans unrelated areas.
- [ ] **No tooling attribution.** Never add "written with an AI", a `Generated-by:` trailer, or a co-authored-by line for a tool. The commit message is a durable record that people read long after the tool is forgotten.

### 4. Draft the PR description

Summarize the whole branch, not just the latest commit.

When the repository contains a pull request template in its supported locations, fill in that template instead of the structure below. If multiple templates are available and none is specified, use the repository's default or most clearly applicable one. Otherwise use this default structure.

- [ ] `## Summary` — two to four sentences on what changed and why.
- [ ] `## Changes` — a bullet list of concrete changes, described by net effect.
- [ ] `## UI notes` — visual or interaction changes a reviewer should look for, such as new states, motion, or responsive behavior. Point out where a screenshot or recording would help.
- [ ] `## Testing` — how the change was or should be verified across viewports, input methods, and reduced-motion settings. Report only checks supported by evidence, and mark anything not performed as pending rather than implying it passed.
- [ ] `## Notes` — only when there is a breaking change, a follow-up, or a deliberate omission.

Apply these regardless of structure.

- [ ] **Never list individual commits.** The reviewer already sees them on the commit tab. Describe the net effect of the branch, not its commit-by-commit history.
- [ ] **No unsupported test claims** such as "ran the test suite". That belongs in CI. Only report manual verification a reviewer cannot find elsewhere, under Testing.
- [ ] **No tooling attribution**, for the same reason as the commit message.

When you update an earlier PR description, rewrite it to describe the full branch as one coherent whole. Do not append a line per commit. Keep it grounded in the actual diff, and do not guess at intent you cannot see in the code.

### 5. Offer to commit and push

After the message is written, offer to act on the active branch. Reuse the branch, upstream, and default-branch state gathered in step 1, and never assume it.

Present only the options that fit that state.

- Commit the already-staged changes with the message.
- Stage all tracked changes, then commit.
- Commit, then push to the existing upstream.
- Commit, then push and set upstream with `git push -u origin <branch>` when no upstream exists.
- Amend the previous commit, offered only when the last commit is not already pushed.
- Do nothing, so the user commits by hand.

Apply these safety rules.

- Committing is reversible, so proceed on the user's selection.
- Push is a shared, hard-to-reverse action, so require an explicit confirmation before it runs, and never force push.
- When the target is the repository default branch, warn and re-confirm before pushing, since a direct push there is often unintended.
- Run the actual git commands and report their real exit result. Never print a success line the command did not produce.

## Writing style

Write every commit body and PR description in plain prose, using only commas and periods to punctuate the sentences you generate. Do not use em dashes, colons, or semicolons. Split a sentence that needs a colon or semicolon into two, and rewrite a dashed aside as a comma or a new sentence. This rule covers the commit body and PR prose you write, not the commit subject line or the structural syntax a tool or convention requires, so the subject line, a `type(scope):` commit prefix, a `PROJ-123:` ticket prefix, a Markdown heading, and anything inside a code span all stay as they are.

Let the text flow end to end. Write each paragraph of a commit body or PR description as one continuous line, and do not hard wrap inside a paragraph. Separate paragraphs with a single blank line only. The subject-line limit is a length cap on one line, not a wrap, so it still applies, and structural syntax that needs its own line breaks, such as Markdown bullets under a PR heading, keeps them.

## Output

Return three sections in this fixed order, with these exact headings. `## Review`, then `## Commit message`, then `## PR description`. Under `## Review`, write one bullet per finding in the form `Severity, path:line, finding`, or write `No issues found.` when the diff is clean. Put the commit message in its own `text` code block, and put the PR description in its own code block. Then present the commit and push offer from step 5.

---

Authored by @planetabhi