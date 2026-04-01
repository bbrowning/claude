---
name: triage-prs
description: Triage open pull requests in the current repository into easy-wins, fix-merges, and needs-review categories using the gh CLI
argument-hint: [filter instructions, e.g. "label:bug" or "exclude label:wontfix" or "only from dependabot"]
disable-model-invocation: true
allowed-tools: Bash, Read, Grep, Glob, Agent, WebFetch, TaskCreate, TaskUpdate, TaskList, TaskGet, TaskOutput, SendMessage, AskUserQuestion
---

# PR Triage

You are a PR triage agent. You analyze all open pull requests in the current
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

Fetch all open PRs with relevant metadata:

```bash
gh pr list --repo <owner/repo> --state open --json number,title,author,labels,isDraft,createdAt,updatedAt,additions,deletions,changedFiles,headRefName,body,reviewDecision,reviews --limit 200
```

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

## Phase 3: Triage Each PR

For each PR, gather enough information to classify it. At minimum, read:
- The PR title, body, and labels
- The diff stats (additions, deletions, changed files)
- Whether it's a draft
- The author (bot vs human, known contributor vs first-time)

For PRs that are not immediately classifiable from metadata alone, fetch the
actual diff to make a judgment:

```bash
gh pr diff <number> --repo <owner/repo>
```

### Category Definitions

Classify every PR into exactly ONE of these three buckets:

---

### 1. EASY-WIN

A PR qualifies as an easy-win if it meets ALL of these criteria:
- **Low risk**: The change is small, well-scoped, and unlikely to break anything
- **Clear intent**: The purpose is obvious from the title/description
- **Minimal review burden**: No architectural decisions, no new APIs, no security implications

Typical easy-wins:
- Documentation fixes (typos, clarifications, formatting)
- Dependency bot auto-upgrades (Dependabot, Renovate) that pass CI
- Small, targeted bug fixes with obvious correctness
- Draft PRs that should be auto-closed (stale, abandoned)
- PRs from banned/spam accounts

**Action for maintainer**: Merge directly, or close with a thank-you note.

---

### 2. FIX-MERGE

A PR qualifies as fix-merge if:
- The contribution has clear value and should be accepted
- But it needs **substantial maintainer modifications** before merging
- The changes needed are clear enough that the maintainer can do them
  faster than requesting changes from the contributor

Typical fix-merges:
- Good idea but code doesn't follow project conventions
- Missing tests but the feature is wanted
- Needs rebasing plus conflict resolution plus minor fixes
- Has a few issues but sending back for revisions would risk contributor abandonment

**Action for maintainer**: Pull the branch locally, make fixes, push with
attribution, and merge.

---

### 3. NEEDS-REVIEW

Everything else. These PRs require deeper investigation and a deliberate
decision. For each needs-review PR, you MUST assign exactly one
**recommendation** from the list below.

---

## Phase 4: Needs-Review Recommendations

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

## Phase 5: Generate the Report

Output a structured triage report with this exact format:

---

### PR Triage Report for `<owner/repo>`

**Date**: YYYY-MM-DD
**Total open PRs**: N (M after filters)

---

#### EASY-WINS (N PRs)

For each PR, one line:
```
- #<number> "<title>" by @<author> -- <brief reason> --> <action: merge/close>
```

---

#### FIX-MERGE (N PRs)

For each PR:
```
- #<number> "<title>" by @<author>
  Issues: <what needs fixing>
  Effort: <low/medium/high>
```

---

#### NEEDS-REVIEW (N PRs)

For each PR:
```
- #<number> "<title>" by @<author>
  Recommendation: <one of the 9 recommendations above>
  Rationale: <2-3 sentences explaining why this recommendation>
  Key concerns: <bullet list of specific issues if any>
```

---

#### Summary

| Category | Count |
|----------|-------|
| Easy-Win | N |
| Fix-Merge | N |
| Needs-Review | N |
| **Total** | **N** |

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

1. **Size heuristics**: PRs with <= 20 changed lines and <= 3 files that are
   docs/config/deps are almost always easy-wins unless they touch security-sensitive files.
2. **Draft PRs**: Drafts older than 30 days with no recent activity are
   easy-wins (close). Recent drafts are excluded from triage unless the user
   specifically asks to include them.
3. **Bot PRs**: Dependabot/Renovate PRs that only change lockfiles or version
   pins are easy-wins. Bot PRs that change source code are needs-review.
4. **First-time contributors**: Be more generous -- prefer Fix-Merge or
   Merge-Fix over Request Changes to avoid discouraging new contributors.
5. **Stale PRs**: PRs with no activity for 90+ days where the branch has
   conflicts should lean toward Retire unless the feature is highly desired.
6. **Security-sensitive changes**: Any PR touching auth, crypto, permissions,
   or input validation is always needs-review, never easy-win.
7. **Large PRs** (500+ lines changed): Always needs-review. Consider whether
   Split-Merge is appropriate.
8. **When in doubt, classify up**: If a PR is borderline between easy-win and
   fix-merge, choose fix-merge. If borderline between fix-merge and
   needs-review, choose needs-review. It's better to over-review than to
   merge something problematic.

## Parallelization

When there are many PRs to triage, use Agent subagents to review multiple
PRs in parallel. Each agent should fetch the diff for its assigned PRs and
return a classification. Coordinate results in the main context.

For repos with fewer than 10 open PRs, sequential processing is fine.
For 10+, batch into groups of 3-5 and parallelize.
