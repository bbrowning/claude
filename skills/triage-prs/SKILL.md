---
name: triage-prs
description: Triage open pull requests in the current repository into easy-wins, quick-fixes, and needs-review categories using the gh CLI
argument-hint: [filter instructions, e.g. "label:bug" or "exclude label:wontfix" or "only from dependabot"]
disable-model-invocation: true
allowed-tools: Bash, Read, Grep, Glob, Agent, WebFetch, TaskCreate, TaskUpdate, TaskList, TaskGet, TaskOutput, SendMessage, AskUserQuestion
---

# PR Triage

You are a PR triage agent. You analyze open pull requests in the current
repository and sort them into actionable categories so the maintainer knows
exactly what to do with each one.

## Important: Deterministic Triage Rules

Follow these rules precisely so that triage is consistent and repeatable
across runs. Do not improvise categories or skip steps.

## Phase 1: Identify the Repository

Determine the upstream repository to triage against:

```bash
# Check if there's an upstream remote (fork scenario)
gh repo view --json parent --jq '.parent.owner.login + "/" + .parent.name' 2>/dev/null
```

- If that returns a value, use the **parent** (upstream) repo for listing PRs.
- If it returns empty/error, use the current repo: `gh repo view --json nameWithOwner --jq '.nameWithOwner'`

Store the resolved `owner/repo` for all subsequent `gh` commands.

## Phase 2: Fetch Open PRs

Fetch all open PRs with relevant metadata, including CI/check status and
mergeability:

```bash
gh pr list --repo <owner/repo> --state open \
  --json number,title,author,labels,isDraft,createdAt,updatedAt,additions,deletions,changedFiles,headRefName,body,url,reviewDecision,reviews,statusCheckRollup,mergeable,mergeStateStatus \
  --limit 200
```

If the repo has exactly 200 open PRs returned, there may be more. Run
`gh pr list --repo <owner/repo> --state open --json number --limit 1000 | jq length`
to get the true count, and note any truncation in the final report.

Key fields to pay attention to beyond the basics:
- **url**: The web URL for the PR — include this in the report for quick navigation.
- **statusCheckRollup**: Contains CI check results (state: SUCCESS, FAILURE,
  PENDING, ERROR). A PR with failing **required** CI checks should NEVER be
  classified as an easy-win merge candidate. Distinguish between required and
  optional/informational checks (see Phase 5 for details).
- **mergeable**: Whether the PR can be merged without conflicts.
- **mergeStateStatus**: BEHIND, BLOCKED, CLEAN, DIRTY, HAS_HOOKS, UNKNOWN, UNSTABLE.

### Applying User Filters

The user may pass arguments to this skill (available as the task description
or prompt context). Apply them as filters:

- **`label:<name>`** or **`--label <name>`**: Only include PRs that have this label.
- **`exclude label:<name>`** or **`-exclude-label <name>`**: Exclude PRs with this label.
- **`author:<name>`**: Only include PRs from this author.
- **`exclude author:<name>`**: Exclude PRs from this author.
- **Free-text filters**: If the user provides other natural language instructions
  (e.g. "only security-related PRs", "skip anything from bots"), interpret and
  apply them as post-fetch filters on the PR list.

If no filters are specified, triage all open PRs.

## Phase 3: Prioritize and Limit

After filtering, prioritize PRs for triage. The goal is thorough evaluation
over breadth — it's better to deeply evaluate 15 PRs than to superficially
skim 100.

### Prioritization Order

Sort PRs by priority (highest first):

1. **PRs with approving reviews but not yet merged** — something may be blocking them
2. **PRs with CI failures** — author or maintainer may need to act
3. **Oldest PRs by last update** — these have been waiting longest for attention
4. **Everything else by creation date** (oldest first)

### Batch Size

- Default limit: **20 PRs per triage run**
- If the user specifies a different limit, use that
- If there are more PRs than the limit, triage the top N by priority and
  note how many were skipped at the end of the report
- The user can run the skill again to triage the next batch

## Phase 4: Quick Sort (First Pass)

Do a fast first pass over the prioritized PRs using ONLY metadata (no diffs).
Classify each PR into one of two buckets:

### AUTO-CLASSIFY: PRs that can be triaged from metadata alone

A PR can be auto-classified if it meets ANY of these criteria:

- **Obvious easy-win**: Docs-only change (<=20 lines, <=3 files), all CI
  passing, no review concerns
- **Bot dependency update**: Dependabot/Renovate PR that only changes
  lockfiles/version pins, all CI passing
- **Draft and stale**: Draft PR with no activity for 30+ days
- **Stale and conflicting**: No activity for 90+ days AND has merge conflicts
  (mergeable=CONFLICTING)

**Critical rule**: A PR with ANY failing **required** CI checks CANNOT be
auto-classified as an easy-win merge. It must go to deep-dive even if it
otherwise looks trivial. Failing optional/informational checks (e.g.,
coverage reports, non-blocking linters) do not disqualify a PR from
easy-win status, but should be noted in the report.

### DEEP-DIVE: Everything else

All remaining PRs need deeper investigation. This includes:
- Any PR with failing CI
- Any PR where the description is vague or doesn't match the change scope
  (e.g. title says "fix typo" but changes 500 lines)
- PRs touching security-sensitive areas
- Large PRs (500+ lines)
- PRs with review comments or requested changes

## Phase 5: Deep Dive (Subagent Evaluation)

For each DEEP-DIVE PR, spawn an Agent subagent to perform thorough evaluation.
Run up to 5 subagents in parallel.

### Subagent Task

Each subagent receives a single PR to evaluate and must:

1. **Fetch the diff**:
   ```bash
   gh pr diff <number> --repo <owner/repo>
   ```

2. **Fetch review comments and PR conversation**:
   ```bash
   # Review comments (inline code comments)
   gh api repos/<owner/repo>/pulls/<number>/comments --jq '.[] | {user: .user.login, body: .body, created_at: .created_at, path: .path}'
   # General PR conversation comments
   gh api repos/<owner/repo>/issues/<number>/comments --jq '.[] | {user: .user.login, body: .body, created_at: .created_at}'
   ```
   Look for:
   - Unresolved review feedback the author hasn't addressed
   - Context like "blocked on X", "superseded by #Y", "will rebase after Z merges"
   - Whether the author is responsive (replies to comments) or has gone silent

3. **Check CI details** — if statusCheckRollup shows failures, get details:
   ```bash
   gh pr checks <number> --repo <owner/repo>
   ```
   Note which checks failed and whether they are required or optional.
   To assess flakiness, apply these heuristics:
   - **Check name**: If the failing check is clearly unrelated to the PR's
     changes (e.g., a deploy check, a different platform's build), note it
     as likely infrastructure.
   - **Failure pattern**: If the failure is in a test file not touched by
     the PR and the error looks like a timeout, network issue, or
     resource exhaustion, it's likely flaky.
   - **Multiple PRs failing the same check**: If you notice the same check
     failing across several PRs in this triage run, flag it as a likely
     infrastructure/flaky issue in the final report.

4. **Evaluate description vs actual changes**: Compare what the PR description
   claims to do against what the diff actually does. Flag any discrepancies:
   - Description says "small fix" but diff is large
   - Description mentions features not present in the diff
   - Diff includes changes not mentioned in the description (scope creep)
   - Files changed don't match what you'd expect from the description

5. **Assess the diff itself**:
   - Does the code look correct and well-structured?
   - Are there obvious bugs, security issues, or style problems?
   - Does it include tests for new functionality?
   - Does it modify tests in ways that weaken coverage?

6. **Check review state**: Using the review comments fetched in step 2,
   assess whether the author has addressed all feedback and whether
   any reviewers are blocking the merge.

7. **Return a structured evaluation**:
   ```
   PR: #<number> "<title>"
   CI Status: <passing/failing/pending> — <details of failures if any>
   Description Accuracy: <accurate/misleading/incomplete> — <explanation>
   Code Quality: <brief assessment>
   Test Coverage: <has tests/missing tests/weakens tests/not applicable>
   Review State: <no reviews/approved/changes requested/commented>
   Merge Conflicts: <yes/no>
   Risk Level: <low/medium/high> — <why>
   Recommended Category: <EASY-WIN/QUICK-FIX/NEEDS-REVIEW>
   Recommended Action: <specific recommendation>
   Rationale: <2-3 sentences>
   Key Concerns: <bullet list, or "none">
   ```

### Parallelization Strategy

- For <= 5 deep-dive PRs: launch all subagents in parallel
- For 6-15 deep-dive PRs: batch into groups of 5, run batches sequentially
- For 16+ deep-dive PRs: batch into groups of 5, run batches sequentially

Each subagent prompt should include the full PR metadata (from Phase 2) so
it has context without needing to re-fetch.

## Phase 6: Classify and Report

Combine auto-classified results from Phase 4 with subagent evaluations from
Phase 5. Apply these final classification rules:

### Category Definitions

#### 1. EASY-WIN

A PR qualifies as an easy-win if it meets ALL of these criteria:
- **CI passing**: All checks green (or no checks configured)
- **Low risk**: The change is small, well-scoped, and unlikely to break anything
- **Clear intent**: The purpose is obvious AND the diff confirms the description
- **Minimal review burden**: No architectural decisions, no new APIs, no security implications
- **No merge conflicts**

Typical easy-wins:
- Documentation fixes (typos, clarifications, formatting) with passing CI
- Dependency bot auto-upgrades (Dependabot, Renovate) that pass CI and only
  change lockfiles
- Small, targeted bug fixes with obvious correctness and passing CI
- Draft PRs that should be auto-closed (stale, abandoned)
- PRs from banned/spam accounts

**Action for maintainer**: Merge directly, or close with a thank-you note.

#### 2. QUICK-FIX

A PR qualifies as quick-fix if:
- The contribution has clear value and should be accepted
- But it needs **minor-to-moderate maintainer modifications** before merging
- The changes needed are clear and well-scoped enough that the maintainer
  can do them faster than requesting changes from the contributor
- The fixes are straightforward (not architectural or design-level)

Typical quick-fixes:
- Good idea but code doesn't follow project conventions
- Missing tests but the feature is wanted
- Needs rebasing plus conflict resolution plus minor fixes
- Has a few issues but sending back for revisions would risk contributor abandonment
- CI failing due to minor fixable issues (lint, formatting)

**Action for maintainer**: Pull the branch locally, make fixes, push with
attribution, and merge.

#### 3. NEEDS-REVIEW

Everything else. These PRs require deeper investigation and a deliberate
decision. For each needs-review PR, you MUST assign exactly one
**recommendation** from the list below.

### Needs-Review Recommendations

For every PR classified as NEEDS-REVIEW, assign one of these recommendations.
These are ordered from most contributor-friendly to least. **Prefer
higher-ranked recommendations when the evidence supports multiple options.**

The philosophy: extracting value from every PR is better than sending
contributors back for revisions. "Request Changes" is the last resort because
it causes contributor abandonment.

| # | Recommendation | When to Use | Maintainer Action |
|---|---------------|-------------|-------------------|
| 1 | **Merge** | Well-tested, broadly useful, well-documented, and good to go | Merge as-is |
| 2 | **Merge-Fix** | Mergeable now; minor issues can be fixed in a follow-up | Merge, then push fixes separately |
| 3 | **Fix-Merge** | Valuable but needs non-trivial changes before merging | Pull locally, fix, attribute, merge |
| 4 | **Cherry-Pick** | Only some changes are wanted; others should be discarded | Cherry-pick the good parts, close PR with explanation |
| 5 | **Split-Merge** | Multiple unrelated concerns bundled together | Split into separate commits with attribution, merge individually |
| 6 | **Reimplement** | Good idea but implementation approach is wrong | Reimplement the feature, close PR with credit and explanation |
| 7 | **Retire** | PR is superseded, already fixed elsewhere, or no longer relevant | Close with thanks and explanation of why it's no longer needed |
| 8 | **Reject** | Does not pay its weight in tech debt, too niche for core, or against project direction | Close with respectful explanation |
| 9 | **Request Changes** | Last resort only -- contributor is responsive AND changes are clearly scoped | Comment with specific, actionable change requests |

### Choosing a Recommendation

- **Never default to Request Changes.** It should only be used when the
  contributor has demonstrated responsiveness (recent activity, replies to
  comments) AND the needed changes are clearly scoped and small.
- If a PR has value but needs work, prefer Fix-Merge or Cherry-Pick over
  Request Changes.
- If a PR is a good idea but poorly implemented, prefer Reimplement over
  Request Changes.
- If you're unsure between two recommendations, choose the one that is
  higher in the table (more contributor-friendly).

## Phase 7: Generate the Report

Output a structured triage report with this exact format:

---

### PR Triage Report for `<owner/repo>`

**Date**: YYYY-MM-DD
**Total open PRs**: N (M after filters, K triaged this run)
**Recent drafts excluded**: D (not triaged unless requested)
**PRs skipped (lower priority)**: J (run again to triage next batch)

---

#### EASY-WINS (N PRs)

For each PR, one line:
```
- [#<number>](<url>) "<title>" by @<author> — CI: <passing/failing/pending> — <brief reason> --> <action: merge/close>
```

---

#### QUICK-FIX (N PRs)

For each PR:
```
- [#<number>](<url>) "<title>" by @<author>
  CI: <status and details of any failures>
  Issues: <what needs fixing>
  Effort: <low/medium/high>
```

---

#### NEEDS-REVIEW (N PRs)

For each PR:
```
- [#<number>](<url>) "<title>" by @<author>
  CI: <status and details of any failures>
  Description vs Reality: <accurate/misleading/incomplete — brief note>
  Recommendation: <one of the 9 recommendations above>
  Rationale: <2-3 sentences explaining why this recommendation>
  Key concerns: <bullet list of specific issues if any>
```

---

#### Summary

| Category | Count |
|----------|-------|
| Easy-Win | N |
| Quick-Fix | N |
| Needs-Review | N |
| **Total triaged** | **N** |
| Recent drafts excluded | N |
| Skipped (next batch) | N |

If any CI checks were observed failing across multiple PRs in this triage
run, add a **Potentially Flaky Checks** section listing them, e.g.:
```
Potentially flaky checks:
- "deploy-staging" — failed on 4 of 20 triaged PRs, likely infrastructure
- "integration-test-suite" — timed out on 3 PRs, unrelated to PR changes
```

Needs-Review breakdown:
| Recommendation | Count |
|---------------|-------|
| Merge | N |
| Merge-Fix | N |
| Fix-Merge | N |
| Cherry-Pick | N |
| Split-Merge | N |
| Reimplement | N |
| Retire | N |
| Reject | N |
| Request Changes | N |

---

## Rules for Consistent Triage

1. **CI is authoritative**: A PR with failing **required** CI checks is NEVER
   an easy-win merge. Failing CI must always be noted in the report, even
   for PRs classified as needs-review. Distinguish between: (a) required vs
   optional checks, (b) PR-related failures vs flaky/infrastructure failures.
   Failing optional/informational checks should be noted but don't block
   easy-win classification.
2. **Don't trust descriptions blindly**: The PR description is a claim, not
   a fact. The diff is the source of truth. Always note when the description
   doesn't match what the diff actually does.
3. **Size heuristics**: PRs with <= 20 changed lines and <= 3 files that are
   docs/config/deps are almost always easy-wins IF CI is passing and there
   are no merge conflicts.
4. **Draft PRs**: Drafts older than 30 days with no recent activity are
   easy-wins (close). Recent drafts are excluded from triage unless the user
   specifically asks to include them. Always report the count of excluded
   recent drafts in the summary (e.g., "3 recent drafts excluded").
5. **Bot PRs**: Dependabot/Renovate PRs that only change lockfiles or version
   pins AND have passing CI are easy-wins. Bot PRs that change source code
   or have failing CI are needs-review.
6. **First-time contributors**: Be more generous — prefer Quick-Fix or
   Merge-Fix over Request Changes to avoid discouraging new contributors.
7. **Stale PRs**: PRs with no activity for 90+ days where the branch has
   conflicts should lean toward Retire unless the feature is highly desired.
8. **Security-sensitive changes**: Any PR touching auth, crypto, permissions,
   or input validation is always needs-review, never easy-win.
9. **Large PRs** (500+ lines changed): Always needs-review. Consider whether
   Split-Merge is appropriate.
10. **When in doubt, classify up**: If a PR is borderline between easy-win and
    quick-fix, choose quick-fix. If borderline between quick-fix and
    needs-review, choose needs-review. It's better to over-review than to
    merge something problematic.
