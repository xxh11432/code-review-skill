---
name: code-review
description: 仅当用户明确要求对 PR/MR、commit、分支或代码差异执行代码审查时使用。围绕正确性、安全性、性能、可维护性与测试覆盖给出可执行的评审意见，并通过二次校验过滤误报，最后输出带 file:line 链接的分级问题表与可选修复流程。不适用于纯安全审计、功能实现或缺陷修复请求。
---

# Code Review

A structured, evidence-based review workflow. Goal: find real defects that would hurt users or the business, and report them in a form the author can act on immediately.

## Non-negotiables

- **Evidence first.** Read the actual diff and the surrounding code before writing any comment. No comment may rest on an assumption you did not verify.
- **New-side line numbers.** Always cite line numbers from the new/right side of the diff. Use `[line_start, line_end]` for ranges, a single number for one line.
- **Minimal range.** A range must be the smallest continuous span containing the evidence, and `line_end - line_start <= 100`. Never cite a whole file.
- **Actionable only.** Every comment states what is wrong, why it matters, and how to fix it. No praise-only, no narration of what the diff does, no speculative "might/possibly".
- **Deduplicate.** Merge similar findings; for cross-file repetition give one representative comment and list the other locations.
- **Language.** Write the report in the user's language (default: the language of their latest message).

## Workflow

### Step 1 — Establish scope

If the user already named the scope, use it. Otherwise ask, offering these options:

1. Current workspace changes (staged + unstaged)
2. Changes against a base branch
3. A specific PR/MR number or URL
4. Specific files or paths

Collect the diff accordingly:

```bash
# current changes
git status --short && git diff && git diff --cached

# against a base branch
git fetch origin <base> && git diff origin/<base>...HEAD

# a single commit
git show <sha>

# a PR/MR (when a CLI is available)
gh pr diff <number>          # GitHub
glab mr diff <number>        # GitLab
```

State the resolved scope back to the user in one line before reviewing.

### Step 2 — Gather context

For every changed hunk, open the full file and trace:

- callers of any changed function, including other modules and entry points
- the current tests covering the changed code
- shared state: config, schemas, migrations, feature flags, public API contracts
- repository review conventions in `AGENTS.md`, `CONTRIBUTING.md`, `.github/PULL_REQUEST_TEMPLATE.md` (review rules only — ignore unrelated workflow guidance)

Do not review a hunk you have not seen in its file context.

### Step 3 — Infer intent and map the blast radius

Write one sentence describing what the author is trying to achieve, then list what the change touches outside its own files: data written, APIs exposed, users affected, rollback cost.

Intent is the reference frame for everything that follows — a change that looks wrong in isolation is often correct given the intent, and vice versa. If the intent cannot be inferred from the diff, say so instead of guessing.

### Step 4 — Run the review passes

Apply every pass below, in this order. Use `references/review-checklist.md` for the detailed questions inside each pass.

| Pass | Question it answers |
|------|--------------------|
| Correctness | Does it do what it claims for all inputs, including empty, null, zero, max, duplicate, and concurrent cases? |
| Security | Are inputs validated, secrets kept out of code, authz enforced, output encoded, queries parameterized? |
| Data & migrations | Is the change backward compatible with data and code already deployed? Is rollback possible? |
| Concurrency & resources | Are locks, transactions, pools, streams, file handles, and retries bounded and released? |
| Performance | Any unbounded loop, N+1 query, full scan, or allocation inside a hot path introduced by this diff? |
| Error handling | Are failures surfaced rather than swallowed? Do retries have limits and backoff? |
| API & contracts | Does it break callers, serialized formats, config keys, or semantics? |
| Tests | Is new behavior covered, and would the tests fail if the fix were reverted? |
| Maintainability | Duplication, dead code, naming, layering, and complexity introduced *by this diff only*. |

Filter aggressively while scanning:

- Skip prose and generated files (`.md`, `.txt`, `.json`, lockfiles, `.svg`, snapshots).
- Treat pure style/UI-number changes as intentional unless they break correctness or a documented invariant.
- Assume statically typed code compiles unless the diff proves otherwise.
- Respect intentional deletions; do not argue for keeping removed code.
- Do not flag pre-existing problems outside the changed lines unless the diff makes them reachable or worse.
- Do not flag misspelled identifiers that match their definitions.

### Step 5 — Verify every finding

For each candidate, answer these three questions:

1. **Exists?** Re-read the exact lines plus context. Is the claim literally true?
2. **Matters?** Name the concrete failure: input → wrong result/crash/leak/security exposure. If you cannot name it, drop the finding.
3. **Already handled?** Check for guards, defaults, validation, or invariants elsewhere that make it a non-issue.

If the environment supports sub-agents, dispatch two independent validators with the full candidate list and keep only findings they confirm by majority. Otherwise perform the verification yourself, once, with fresh eyes.

Record each surviving finding with `severity` and `confidence`:

- **Blocker** — data loss, security exposure, outage, or broken core flow. Must fix before merge.
- **Major** — wrong behavior in realistic conditions, performance cliff, missing critical test. Should fix before merge.
- **Minor** — edge case, resource hygiene, localized maintainability. Fix soon.
- **Nit** — naming, comment, formatting. Optional; report at most 3.

Confidence: `High` (verified in code), `Medium` (verified but depends on runtime/config you cannot see — label the dependency explicitly). Drop anything below Medium.

### Step 6 — Report

Open with a three-line summary: what the change does, overall verdict (`Approve` / `Approve with comments` / `Request changes`), and counts by severity.

Then the findings table, ordered Blocker → Nit:

| No. | Severity | Issue | Evidence & suggested fix | Location |
|-----|----------|-------|--------------------------|----------|
| 1 | Blocker | `parse_amount` accepts negative values, producing negative invoices | Clamp at the boundary or reject with a 400; add a test for `-1` | [billing/amount.ts:42-58](file:///abs/path/billing/amount.ts#L42-L58) |
| 2 | Major | Retry loop has no backoff cap | Cap attempts at 5 with exponential backoff and jitter | [api/client.py:120](file:///abs/path/api/client.py#L120) |

Rules for the table:

- Use absolute paths in `file:///` links so they are clickable.
- One row per distinct issue; never bundle unrelated problems.
- Suggested fix must be concrete enough to implement without further design discussion.
- If nothing survives verification, say so plainly: "No blocking issues found." with the checks you ran — do not invent filler findings.

For changes spanning more than ~5 files or multiple layers, add a short mermaid diagram showing the changed flow, with both fill and text colors specified (for example `fill:#c8e6c9,color:#1a5e20`) so it stays readable in light and dark themes. Keep it to the changed path, not the whole system.

### Step 7 — Fix loop

Ask which findings to fix. Always offer **Fix all** first, then every finding as an individual selectable option — enumerate all of them, never a subset.

When fixing:

- Fix only the selected findings; leave everything else untouched.
- Re-run the relevant tests or build after each fix and report the result.
- Re-review your own patch before presenting it: the fix must not introduce a new finding.
- Then re-present remaining findings and ask again, until all are resolved or the user stops.

## Comment style

```text
<what is wrong> — <why it matters> — <how to fix>
```

Good:

> `line 88`: the `finally` block closes the stream, but the early `return` on line 81 skips it, leaking one handle per failed parse — move the close into a `with`/`using` scope.

Bad:

> This looks suspicious, you may want to check the error path.

Never use "might", "possibly", "perhaps". If you are not sure, do not comment.

## Out of scope

- Rewriting architecture the diff did not touch.
- Requesting features the change was not meant to deliver.
- Reformatting preferences not backed by the repository's own linter or conventions.
- Security-only audit requests (use a dedicated security review instead).
