---
name: review-changes
description: Use when reviewing a changeset, including GitHub PRs, local branches, commit ranges, working tree changes, docs, config, tests, CI/CD, infrastructure, migrations, or other reviewable work; autonomously posts formal GitHub PR reviews (comments/approve/request changes) and supports read-only review for non-PR targets.
---

# Review Changes

Use this skill for substantive reviews of changesets. A changeset may be a GitHub pull request, a local branch, a commit range, staged or unstaged working tree changes, a set of files, or other work that could reasonably be reviewed before merge. The goal is to produce high-signal review feedback, pressure-test it, and — for GitHub PRs — post it autonomously as a formal review.

## Core Principles

- Treat review as a bug/risk finding exercise, not a summary exercise.
- Review everything that could be in a PR: code, tests, docs, config, CI/CD, infrastructure, migrations, schemas/contracts, dependencies, and generated artifacts when intentionally included.
- Prioritize correctness, security, data integrity, deployment/runtime breakage, CI/validation status, tests, documentation, commit atomicity, observability, and rollback risk.
- Minor nits are allowed, but clearly label them as non-blocking.
- Always do a second pass before presenting or posting findings.
- This skill is for posting reviews to GitHub PRs and should operate autonomously. For GitHub PR targets, always post the review (inline comments and/or an overall review with `COMMENT`, `APPROVE`, or `REQUEST_CHANGES`) directly. Do not ask whether to post, what review mode to use, or for approval of a preview — decide the disposition from the Severity Guidance and post it.
- Never expose secrets. If a secret appears in a file or log, report that secret material was exposed without repeating the value.
- Use `gh` for GitHub operations when available.
- Prefer line comments for findings tied to exact changed diff lines.
- Use the main PR comment for cross-cutting findings, items outside the diff, review stance, and deduplicated summary.
- For GitHub PRs, review existing comments, reviews, and author responses before deciding what feedback to add.
- Do not approve a PR if unresolved blocker-level findings remain.
- The only question this skill should ever ask the user is whether to include low/nit-priority findings in the posted review, and only ask that when genuinely unsure whether they add value (e.g., a large number of nits that could clutter the review, or nits of ambiguous relevance). When there are only a few clear nits, or nits are clearly worth surfacing, include them without asking.

## User Interaction Tools

The only question this skill should ever ask the user is the low/nit-priority inclusion question described above, and only when genuinely unsure. Do not use the question tool (or ask in chat) for target selection, ticket context, review mode, posting intent, disposition, or preview approval — decide autonomously using the guidance in this skill and proceed.

- In OpenCode, use `question`.
- In Claude Code, use `askuserquestion` or the equivalent available ask-user tool.
- If no structured question tool is available, ask directly in chat and wait for the answer before continuing.

Ask concisely with recommended options when this rare question is warranted.

## Review Modes

### Formal GitHub Review (default for PR targets)

Post inline comments and/or an overall GitHub PR review with one of:

- `COMMENT`
- `APPROVE`
- `REQUEST_CHANGES`

Use automatically whenever the target is a GitHub PR. Determine the disposition from the Severity Guidance section and post it without asking. This is the default and expected outcome for any GitHub PR review — do not stop short of posting.

### Read-Only Review

Inspect the changeset and return findings, change summary, 4C evaluation, and draft review body in chat. Do not post.

Use only when:

- The target is not a GitHub PR (a local branch, commit range, working tree change, or file set — there is nothing to post to).
- The user explicitly asks for a read-only/no-post review of a PR.

## Required Workflow

### 1. Determine Review Target

Determine what is being reviewed before gathering context. Most users will specify the target in the request; if they do not, ask.

Supported targets include:

- GitHub PR URL.
- PR number.
- `owner/repo#123`.
- Local branch name.
- Current local branch.
- Commit range.
- Staged changes.
- Unstaged working tree changes.
- Specific files or directories, when explicitly requested.

If the target is missing or ambiguous, ask the user to choose the review target.

Recommended target options:

- `Current Branch`: Review current branch against the repository default branch.
- `GitHub PR`: Review a PR URL, number, or `owner/repo#123`.
- `Local Branch`: Review a named local branch against the repository default branch unless another base is provided.
- `Working Tree`: Review staged and/or unstaged local changes.
- `Commit Range`: Review an explicit commit range.
- `Files Or Directories`: Review specific paths.

For local branch reviews, default to comparing against the repository default branch unless the user asks for a different base. Determine the default branch from GitHub metadata or local git refs when possible. If unable to determine it, ask the user for the base branch.

Recommended commands for default branch discovery:

```bash
gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'
git symbolic-ref refs/remotes/origin/HEAD --short
git remote show origin
```

Do not mutate the user's branch while determining the target. Do not checkout, rebase, reset, stash, or commit unless explicitly requested and approved.

### 2. Gather Ticket Or Acceptance Criteria

Before reviewing, check whether the user already provided ticket context, acceptance criteria, issue links, product requirements, or a PR description. Do not ask the user for this — use what is available (PR body, linked issues, commit messages) and proceed. If no ticket context exists, review against code, tests, docs, and stated intent only, and do not block the review.

### 3. Gather Changeset Context

Gather metadata, files, checks, comments, and diffs appropriate to the target type.

#### GitHub PR Target

Recommended commands:

```bash
gh pr view <pr> --json number,title,author,baseRefName,headRefName,headRefOid,body,commits,files,additions,deletions,reviewDecision,statusCheckRollup,comments,reviews
gh pr diff <pr> --name-only
gh pr checks <pr>
gh pr diff <pr>
gh api repos/<owner>/<repo>/pulls/<number>/comments
gh api repos/<owner>/<repo>/issues/<number>/comments
```

Also inspect relevant changed files directly where needed:

```bash
gh api -H "Accept: application/vnd.github.raw" \
  "repos/<owner>/<repo>/contents/<path>?ref=<headRefName>"
```

#### Local Branch Target

Default to comparing the branch against the repository default branch unless the user specified another base.

Recommended commands:

```bash
git status --short
git branch --show-current
git diff --name-only <base>...<branch>
git diff <base>...<branch>
git log --oneline <base>..<branch>
```

Use `HEAD` for `<branch>` when reviewing the current branch.

#### Working Tree Target

Clarify whether to review staged changes, unstaged changes, or both when unclear.

Recommended commands:

```bash
git status --short
git diff --cached --name-only
git diff --cached
git diff --name-only
git diff
```

#### Commit Range Target

Use the exact range the user provides. If the range is ambiguous, ask.

Recommended commands:

```bash
git diff --name-only <range>
git diff <range>
git log --oneline <range>
```

#### File Or Directory Target

Read the requested files and inspect nearby context. If the user expects a diff-based review, ask for the base or range.

Use line-numbered reads when possible so findings have precise references.

### 4. Review Existing PR Feedback

For GitHub PR targets, inspect existing review comments, issue comments, review decisions, and author responses before doing the main review. Skip this step for non-PR targets unless an associated PR is discovered.

Use existing PR feedback to determine:

- Whether this is the first substantive review, a follow-up review, or a re-review after author changes.
- Which findings others already raised.
- Which existing threads or comments appear unresolved.
- Whether the author responded to or addressed prior feedback.
- Whether new commits likely addressed prior comments.
- Whether proposed findings would duplicate existing feedback.
- Whether prior blocker-level concerns still affect the current diff.

Recommended commands:

```bash
gh pr view <pr> --json comments,reviews,reviewDecision,statusCheckRollup,commits,headRefOid
gh api repos/<owner>/<repo>/pulls/<number>/comments
gh api repos/<owner>/<repo>/issues/<number>/comments
```

When richer thread state is needed, use the GitHub GraphQL API to inspect review threads and resolution state:

```bash
gh api graphql -f query='<query for pullRequest reviewThreads { nodes { isResolved comments { nodes { author { login } body path line createdAt } } } } }>'
```

Avoid reposting the same issue unless the prior feedback is incomplete, incorrect, unresolved and needs escalation, or the current diff reintroduced the issue. If existing feedback already covers a concern, acknowledge it in the review context instead of duplicating it.

### 5. Understand Changeset Intent

Extract:

- What the changeset claims to do.
- What systems it touches.
- Whether it affects infrastructure, deployment, auth, persistence, billing, background jobs, data migrations, third-party APIs, permissions, docs, or secrets.
- Whether production is affected immediately or later.
- Which checks ran and what they actually validated.
- Current CI status, including failed, pending, skipped, missing, or irrelevant checks.
- Whether documentation, migration notes, runbooks, API docs, user docs, or setup instructions should change.
- Whether commits are atomic, reviewable, and organized as coherent vertical slices.
- For PRs, where the changeset sits in the review lifecycle and whether prior feedback has been addressed.

If intent is unclear, infer it from the PR title/body, commits, and diff, and note the inferred intent in reviewer notes rather than asking. If the changeset is large, focus first on files with runtime, deployment, security, data, or compatibility impact.

### 6. Fan Out Specialist Review

When agents or subagents are available, fan out focused review work after gathering context. If agents are not available, perform these passes manually.

Recommended specialist passes:

- `Security And Data Integrity`: Secrets, auth, permissions, injection, unsafe deserialization, data loss/corruption, migration safety, rollback risk.
- `Test Coverage`: Whether changed behavior is exercised by unit, integration, e2e, CI, migration, or infrastructure validation.
- `Repo Compatibility`: Whether the changes fit existing architecture, conventions, dependency versions, generated code workflows, public APIs, consumers, deployment paths, and runtime assumptions.
- `Ticket Validation`: Whether the changes satisfy ticket context and acceptance criteria, and whether anything requested is missing or contradicted.
- `Documentation`: Whether user-facing docs, API docs, setup docs, operational docs, migration notes, examples, changelogs, or runbooks need updates; whether existing docs became stale.
- `Commit Atomicity`: Whether commits are focused, independently reviewable, correctly ordered, and free of fixup/WIP/noise commits or unrelated changes.
- `Review Lifecycle`: For PRs, whether existing comments already cover candidate findings, whether author responses addressed prior feedback, whether unresolved threads remain relevant, and whether this is a first review or follow-up.

Each specialist pass should return:

- Confirmed findings.
- Weakened or speculative findings.
- Files or areas inspected.
- Relevant evidence.
- Confidence level.

Do not treat specialist output as final. Reconcile duplicates, pressure-test claims, and discard findings that do not survive scrutiny.

### 7. Build High-Level Change Summary

Build a concise bulleted summary of the main changes for the reviewer. Include non-code changes such as docs, config, CI/CD, infrastructure, dependency updates, migrations, generated artifacts, and tests when present.

Use this summary to confirm scope and catch mismatches between stated intent and actual changes. If the summary reveals ambiguity, note it as an open question in reviewer notes rather than asking the user.

### 8. Initial Review Pass

Identify candidate findings.

For each candidate finding, capture:

- Severity: critical, high, medium, low, nit.
- File and line if possible.
- The concrete failure mode.
- Why it matters.
- What would fix or mitigate it.
- Evidence from code, diff, logs, docs, CI, ticket criteria, or existing repo behavior.
- Whether it is introduced by this changeset or appears pre-existing.
- Whether existing PR feedback already covers it.

Prefer findings that answer:

- Can this break deploys?
- Can this break runtime behavior?
- Can this expose secrets or broaden permissions?
- Can this corrupt or lose data?
- Are required checks failing, pending, skipped, or missing?
- Are tests/CI missing for the changed behavior?
- Does rollback still work?
- Are generated resources/configs tied to the wrong environment?
- Are there concurrency, timeout, idempotency, pagination, retry, or cold-start issues?
- Are inputs validated and are injection risks avoided?
- Are docs required because setup, usage, API behavior, operations, or migration behavior changed?
- Does the implementation satisfy the ticket or acceptance criteria?
- Are commits atomic, coherent, and aligned with the repo's commit conventions?
- Has existing PR feedback already raised or resolved this concern?

Avoid:

- Style-only comments unless the user requested them.
- Speculative findings without a concrete failure path.
- Repeating the same issue in multiple places unless each instance matters.
- Posting or presenting pre-existing issues as if this changeset introduced them.
- Duplicating existing PR review feedback unless the duplicate is intentional and adds value.

### 9. Evaluate The 4Cs

Evaluate the changeset across the 4Cs. Keep this separate from severity-ranked findings so it provides review context without diluting blocker-level issues.

- `Correctness`: Does the changed behavior work as intended, including edge cases, errors, security, data integrity, and operational behavior?
- `Completeness`: Does it satisfy ticket criteria, stated intent, required tests, required docs, migrations, compatibility, and rollout needs?
- `Conciseness`: Is the change appropriately scoped, with atomic commits and without avoidable complexity, duplication, unrelated edits, or unnecessary dependencies?
- `Clarity`: Is the implementation understandable, with clear names, readable tests, useful docs, and maintainable structure?

Use clear ratings such as `strong`, `acceptable`, `needs work`, or `unclear`, with short evidence for each. If a 4C assessment depends on missing context, label it `unclear` rather than asking the user.

### 10. Pressure-Test Pass

Before showing findings to the user or posting them, re-check every candidate finding.

For each finding, classify it as:

- `confirmed blocker`: clear bug/security/deploy risk that should block.
- `confirmed non-blocking`: valid but not a blocker.
- `needs context`: plausible, but depends on external configuration or intent.
- `weakened`: partially valid but severity should be reduced.
- `drop`: does not survive scrutiny.

Pressure-test using:

- Actual workflow logs, if relevant and available.
- CI status, check results, and failing check logs.
- GitHub Actions reusable workflow semantics.
- Existing repo conventions.
- CI workflow path filters and job contents.
- Documentation for platform behavior, if needed.
- Current PR comments and author notes.
- Base branch behavior, to avoid flagging pre-existing issues as introduced.
- Ticket criteria and explicit user-provided requirements.
- Specialist agent findings and counter-evidence.
- Commit history, to distinguish patch-level issues from history hygiene issues.
- Existing PR comments, review threads, author replies, and later commits.

When a finding weakens, say so plainly. Do not keep weak findings as blockers.

### 11. Decide Comment Placement

Split final feedback into two buckets when the target is a GitHub PR. For non-PR targets, produce the same content as chat output instead of posting.

#### Inline Comments

Use inline comments when:

- The issue maps to a changed line in the PR diff.
- The comment can be understood locally.
- The comment asks for a concrete change or clarification.

Inline comment format:

```md
<concise issue statement>

<why it matters / concrete failure mode>

<suggested fix or question>
```

When the finding has a small, concrete code fix (e.g. reordering a conditional, guarding a null check, swapping an operator), include the fix as a short fenced code snippet in the inline comment rather than only describing it in prose. For example:

```md
Here is a potential fix:

​```ts
if (!x) {
	...
}
​```
```

Keep the snippet minimal — just the changed lines or the smallest coherent block, not a full-file diff. Only include a code snippet when the fix is genuinely small and unambiguous; for larger or more involved changes, describe the fix in prose instead.

Keep inline comments focused.

#### Reviewer Notes Versus Author-Facing Review

Keep internal reviewer analysis separate from PR feedback intended for the author.

Use reviewer notes for:

- Existing review context and PR lifecycle stage.
- CI/validation status and whether checks cover the changed behavior.
- High-level change summary.
- 4C evaluation.
- Ticket or acceptance criteria validation.
- Commit atomicity and history hygiene.
- Specialist pass notes.
- Weakened or dropped findings.
- Duplicates intentionally avoided because prior feedback already covers them.
- Recommended review mode and disposition.

Use the author-facing review body for concrete PR feedback:

- Confirmed blockers or non-blocking findings that need author action.
- Cross-cutting issues that do not map cleanly to a changed line.
- Missing CI/test/doc coverage when actionable.
- Deployment sequencing or rollback concerns when actionable.
- Direct questions that need author clarification.
- Final stance in author-facing language.

Do not dump reviewer notes into the PR comment. The author-facing review should read like practical PR feedback to the author, not an analysis report to the reviewer.

The author-facing review body should avoid repeating inline details, and should not include a section that summarizes or lists what the inline comments cover — that is redundant with the inline threads themselves. Limit the body to cross-cutting findings, direct questions, and the overall stance.

### 12. Decide Disposition And Handle Nits

For GitHub PR targets, decide the disposition autonomously from the Severity Guidance section — do not ask the user:

- `REQUEST_CHANGES`: Confirmed blocker-level findings remain, prior blocker feedback is still unresolved, or required CI is failing because of the changeset.
- `APPROVE`: No unresolved blocker-level or important correctness issues remain (including unresolved prior blocker feedback), and required CI is not failing or pending.
- `COMMENT`: Findings are non-blocking/informational, need author clarification, or CI is pending/inconclusive without confirmed blockers.

Post inline comments alongside the overall review whenever findings map to specific diff lines.

Nits handling: by default, include low/nit-priority findings in the review (as inline comments or a short non-blocking note) without asking. Only invoke the question tool to ask whether to include them when genuinely unsure they add value — for example, a large volume of nits that could clutter the review, or nits of ambiguous relevance to this repo's conventions. This is the only situation in which this skill should ask the user anything.

Handle security-sensitive findings by summarizing them privately in the review body (no exploit details, no secret values, no live payloads) rather than omitting them or asking whether to post them.

### 13. Build The Review

Generate the complete review content that will be posted:

- Target PR and reviewed `headRefOid`.
- Formal disposition: `COMMENT`, `APPROVE`, or `REQUEST_CHANGES`.
- Author-facing review body.
- Inline comments, including path, line, side, and body.
- Any line-mapping uncertainty or comments that could not be placed inline (fold these into the main review body instead of dropping them).

Re-fetch the current PR `headRefOid` before building the review. If the PR head changed since context was gathered, re-gather the diff and comments for the new head before proceeding — do not ask the user, just refresh and continue.

### 14. Posting Procedure

Post the review directly. Do not wait for approval or a preview confirmation — this skill posts autonomously.

Before posting:

- Get current `headRefOid` and confirm it matches what the review was built against; if it changed, refresh context and rebuild the review for the new head.
- Prefer GitHub line comments on changed diff lines.
- If line mapping fails, fold that finding into the main review body instead of stopping to ask.
- Never include secret values.

Recommended command:

```bash
gh pr view <pr> --json headRefOid --jq '.headRefOid'
```

Post inline comments with:

```bash
gh api -X POST "repos/<owner>/<repo>/pulls/<number>/comments" \
  -f commit_id="<headRefOid>" \
  -f path="<path>" \
  -F line=<line> \
  -f side="RIGHT" \
  -f body="<comment body>"
```

Submit the formal review with `gh pr review` using the disposition decided in step 12:

```bash
gh pr review <number> -R <owner>/<repo> --comment --body "<body>"
gh pr review <number> -R <owner>/<repo> --approve --body "<body>"
gh pr review <number> -R <owner>/<repo> --request-changes --body "<body>"
```

### 15. After Posting

Report:

- What was posted.
- Links to posted comments or review.
- Anything that failed to post.
- Any findings intentionally kept as draft-only.
- Whether the PR head changed during review or posting.

## Severity Guidance

### Blocking / Request Changes Candidates

Use blocker-level language for:

- Runtime failures on normal paths.
- Deployment failures or broken promotion/cutover.
- Secret exposure or material permission expansion.
- Data corruption/loss risks.
- Missing validation for user-controlled input that can create injection/security issues.
- Missing CI for newly added critical infra code.
- Tests passing while not exercising the changed behavior.
- Required docs or migration instructions missing when omission can cause failed deploys, broken setup, unsafe operations, or incorrect user behavior.

### Non-Blocking Candidates

Use non-blocking language for:

- Maintainability concerns without immediate failure.
- Hard-coded values in staging-only paths when production is unaffected.
- Cleanup/reporting drift that does not break execution.
- Documentation gaps that do not change behavior or operational safety.
- Naming or style issues.

## Security And Secret Handling

Always proactively look for:

- Secrets committed to files.
- Secrets printed in CI logs.
- Generated secrets not masked with `::add-mask::`.
- Over-broad IAM, OIDC, GitHub token, cloud, or deploy permissions.
- Public endpoints relying only on weak/shared keys.
- Missing auth, missing CORS restrictions, or unintended anonymous access.
- Shell injection through unquoted PR/user-controlled values.
- Unsafe deserialization or command execution.
- Missing validation for user-controlled input.

If secret material is discovered:

- Do not repeat it.
- State that secret material was exposed without repeating the value.
- Recommend rotation if the value may be live.
- Recommend masking and safer handling.
- Post this finding as a summary in the review body without exposing the secret or exploit path — do not ask before posting, just keep the public comment safe.

## Documentation Review Checklist

When behavior, setup, operations, APIs, migrations, or configuration change, inspect:

- README and setup instructions.
- API docs and generated contract docs.
- User-facing docs, examples, and screenshots.
- Operational runbooks and deployment notes.
- Migration guides, rollback notes, and data backfill instructions.
- Environment variable documentation and `.env.example` placeholder names.
- Changelog or release notes when the repo uses them.
- Comments or docs that become stale because of the changeset.

Documentation findings should explain who would be misled or blocked by the missing/stale docs.

## Commit Atomicity Review Checklist

When the target has commit history, inspect whether commits follow atomic commit principles.

Look for:

- Each commit represents one coherent, reviewable change.
- Refactors, mechanical changes, and feature behavior are separated when practical.
- Tests and docs are included in the same vertical slice as the behavior they validate or describe.
- Commit order tells a logical implementation story.
- No `wip`, `fixup`, `oops`, revert-noise, or follow-up commits should be cleaned up before review.
- No unrelated files or opportunistic edits are mixed into a commit.
- Commit messages follow repository conventions, such as Conventional Commits when required.
- Each commit can reasonably pass validation on its own when the repository expects that standard.

Recommended commands:

```bash
git log --oneline <base>..<branch>
git diff-tree --stat --oneline <commit>
git show --stat --oneline --name-status <commit>
```

Treat commit atomicity findings as review hygiene unless the history creates practical risk, such as making rollback, bisect, release notes, or review materially harder. If the final diff is correct but the history should be cleaned up, recommend reslicing instead of blocking on code behavior.

## Review Lifecycle Checklist

When reviewing a GitHub PR, inspect existing feedback so the review fits the PR's current lifecycle.

Look for:

- Whether this is the first substantive review, a follow-up review, or a re-review after author changes.
- Existing review comments that already cover candidate findings.
- Unresolved review threads or comments that still apply to the current diff.
- Author comments that explain intent, reject prior feedback, or claim an issue was fixed.
- New commits after prior reviews that may have addressed earlier comments.
- Prior approvals, requests for changes, or review decisions that should affect the recommended disposition.
- Feedback that should not be duplicated because it is already clear and visible.
- Feedback that should be escalated because it remains unresolved after prior discussion.

Use lifecycle context to calibrate tone and disposition. A first review can introduce issues directly; a follow-up review should focus on what changed, what remains unresolved, and whether earlier feedback was addressed.

## CI And Validation Checklist

For GitHub PRs, inspect CI status before recommending a disposition.

Look for:

- Required checks that are failing, pending, skipped, cancelled, or missing.
- Non-required checks that reveal real risk.
- Checks that are green but do not exercise the changed behavior.
- Path filters or workflow conditions that skipped validation unexpectedly.
- Failing logs that confirm or contradict candidate findings.
- Whether docs, infra, migrations, generated code, or dependency changes received appropriate validation.

For local branches or working tree reviews, identify the smallest relevant validation from repo docs, package scripts, build files, CI config, or existing conventions. Do not run expensive or destructive validation unless the user asked for it or explicitly approves.

Use CI context carefully. Failing required CI should usually prevent an approval recommendation, but passing CI does not prove correctness if the changed behavior is not covered.

## GitHub Actions Review Checklist

When workflows change, inspect:

- Trigger events and branch filters.
- `pull_request` vs `pull_request_target` risks.
- `permissions`.
- Environment usage and environment secrets.
- Reusable workflow secret passing semantics.
- Path filters and whether changed files actually trigger relevant CI.
- Concurrency groups.
- Whether generated secrets or heredocs appear in logs.
- Whether deployment jobs run on PRs, pushes, or only after merge.
- Whether outputs are wired correctly.
- Whether hard-coded ARNs, URLs, accounts, regions, or role names are intentional.

## Infrastructure Review Checklist

When IaC or deployment changes, inspect:

- Least privilege.
- Resource naming and environment isolation.
- Staging/prod separation.
- Secrets source and runtime loading.
- Rollback path.
- State/backing service migration.
- Generated outputs and how consumers receive them.
- Timeouts, memory, concurrency, cold starts, idempotency, retries.
- Build context and ignored files.
- CI validation: format, lint, typecheck, unit tests, synth/plan.
- Whether production is affected now or only in a follow-up.

## Output Templates

### Read-Only Review Output

```md
**Main Changes**

- <high-level change>
- <docs/tests/config/infra/migration change if relevant>

**Existing Review Context**

- Review stage: <first review/follow-up/re-review after changes/not applicable>
- Existing feedback considered: <summary>
- Duplicates avoided: <summary>
- Prior feedback still relevant: <summary>

**Findings**

- Critical: <issue> (`path:line`)
- High: <issue> (`path:line`)
- Medium: <issue> (`path:line`)

**4C Evaluation**

- Correctness: <rating> - <evidence>
- Completeness: <rating> - <evidence>
- Conciseness: <rating> - <evidence>
- Clarity: <rating> - <evidence>

**Ticket / Acceptance Criteria**

- <criteria satisfied or missing>

**CI / Validation**

- Status: <passing/failing/pending/skipped/not run/unknown>
- Failed checks: <summary>
- Missing or skipped expected checks: <summary>
- Coverage of changed behavior: <adequate/gap/unclear>

**Specialist Pass Notes**

- Security/Data Integrity: <summary>
- Test Coverage: <summary>
- Repo Compatibility: <summary>
- Documentation: <summary>
- Commit Atomicity: <summary>
- Review Lifecycle: <summary>

**Weakened/Dropped After Second Pass**

- <finding>: softened/dropped because <reason>.

**Questions**

- <question, only if the nits question from step 12 applies; otherwise omit>
```

### Author-Facing Formal Review Body

Write a PR-specific review body for the author. Avoid canned greetings, internal scoring, reviewer-only analysis, and exhaustive summaries. Make the feedback concrete, actionable, and proportional to the selected disposition.

Use this shape for request-changes reviews:

```md
Requesting changes because <specific blocker>. <Describe the concrete failure mode and impact in one or two sentences>.

Additional notes:

- <cross-cutting actionable issue, if any>
- <missing validation/docs/deployment concern, if any>

Overall: <what needs to change before this is ready>.
```

Use this shape for comment reviews:

```md
I left a few comments/questions worth considering. Nothing from this pass looks blocking, but <area> may need confirmation or follow-up.

Additional notes:

- <non-blocking follow-up or question, if any>

Overall: <short PR-specific stance>.
```

Use this shape for approvals:

```md
This looks good to me. <Mention any important non-blocking context, if relevant>.

Non-blocking follow-ups:

- <follow-up, if any>

Overall: <short approval rationale tied to the PR>.
```

Omit empty sections.

### Posted Review Summary

After posting, report exactly what was submitted.

```md
**Posted Review**

- Target: <owner/repo#number>
- Head SHA: <headRefOid>
- Disposition: <COMMENT/APPROVE/REQUEST_CHANGES>
- Link: <review URL>

**Main Review Body**

<exact body posted>

**Inline Comments**

- `<path>:<line>` `<side>` — <link>

  <exact inline comment body>

**Not Posted**

- <finding or note intentionally kept out of GitHub, e.g. omitted nits, and why>

**Sensitive Findings**

- <private handling summary without secret values or exploit details>
```

## Important Constraints

- Do not mutate the user's branch while reviewing.
- Do not checkout, rebase, reset, stash, or commit unless explicitly needed.
- Do not commit review artifacts.
- For GitHub PR targets, always post the review — do not stop to ask whether to post, what mode to use, or for preview approval.
- Do not overstate speculative issues.
- Do not approve a PR if unresolved blocker-level findings remain.
- Do not request changes when the decided disposition is `COMMENT`.
- Do not expose secret values.
- Do not ask the user anything except the low/nit-priority inclusion question, and only when genuinely unsure.
- Do not post sensitive security details (secret values, exploit paths) publicly — summarize them privately in the review instead.
