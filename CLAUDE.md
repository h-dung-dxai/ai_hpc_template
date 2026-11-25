# CLAUDE.md — Total DCIM (DRAFT)

## Ticket rules

- A ticket must contain all 6 items below. If any item is missing, ask the person who assigned the work. Do not guess.

| Item | What to write | Example |
|---|---|---|
| Title | `[Task ID][Priority] What to do`. One short sentence starting with a verb | `[A17-01][P0] Redraw documents #1651/#1660 as data flow diagrams` |
| Type | Choose in the *Tracker* field: **Feature** (new or changed functionality) or **Bug** (fix). Work that changes code uses only these 2 types | Feature / Bug |
| Assignee | Exactly one person, set in the *Assignee* field. If not decided, write "waiting for PM assignment". Do not assign other people yourself | Dung |
| Purpose and requirements | Why it is needed; what to do, how far the scope goes, what is **not** done | Revise the documents to center on diagrams. No further in-depth investigation |
| DoD | A list of checkable conditions; each line can be answered "met / not met" | Has diagram + legend + sample data |
| Reference documents | Links to where the request came from: meeting minutes, spec, related tickets, customer documents, design files | Internal meeting minutes 17/11; #1651, #1660 |

Template to paste into the ticket Description:

```text
■ Assignee
<Name> (or: waiting for PM assignment)

■ Purpose
<Why this work is needed, 1–3 sentences>

■ Requirements
- <Task 1>
- <Task 2>
- Out of scope: <what will not be done>

■ Definition of Done (DoD)
- [ ] <Condition 1, checkable>
- [ ] <Condition 2, checkable>

■ Reference documents
- <Meeting minutes / spec / customer document: name + link>
- Related tickets: #<number>

■ How to reproduce (Bug only)
- Environment: <staging / production / local, version>
- Steps: <1. … 2. …>
- Expected result: <…>
- Actual result: <…, with log / screenshot>
```

- **DoD must be measurable.** "Code works well" is not a DoD. "Tests pass, screenshot of screen X attached, data A matches B" is a DoD.
- **Reference documents must be openable.** Give a link or file path, plus page or section number for long documents.
  Do not write something vague like "per spec".
- **A bug must have reproduction steps.** If missing, send it back to the reporter.
- Due date, priority and parent ticket go in the matching Redmine fields. Do not repeat them in the description.

## Workflow

Applies to every code change in the Total DCIM project. Each task goes through 8 steps in order.
Between two steps there is a gate: until its conditions are met, you cannot move to the next step.

### Overview

| Step | Who | Required output | Condition to move on |
|---|---|---|---|
| 01 Receive request | Assignee | Redmine ticket following the template | Ticket has all 6 items: title, type (Feature / Bug), assignee, purpose and requirements, DoD, reference documents |
| 02 Analysis | Assignee | Clear understanding of scope, affected parts, risks | All unclear points have been asked and answered |
| 03 Estimate | Assignee, approved by PM | Estimated hours in the ticket | PM agrees with the estimate |
| 04 Code + local checks (hard requirement) | Assignee | Own branch, tests pass | All local checks and DoD pass |
| 05 Create PR + evidence (hard requirement) | Assignee | PR with full evidence | PR description complete, CI green |
| 06 Review ≥ 2 approvals (hard requirement) | At least 2 reviewers | 2 approvals | No unresolved comments |
| 07 Merge | Assignee | Code on the main branch | All merge conditions met |
| 08 Deploy | Assignee, PM approves production | New version running on staging, then production | Smoke tests pass, ticket updated |

### Using AI (Claude Code)

| AI allowed | No AI, humans only |
|---|---|
| 02 Analysis | 06 Approve PR |
| 04 Implement code | 07 Merge code |
| Bug fixing | 08 Deploy |

- Approve, merge and deploy need a human in control to prevent mistakes.
- Claude does not run approve, merge or deploy commands (e.g. `gh pr review --approve`, `gh pr merge`, deploy scripts),
  even when asked. Remind the user to do it themselves.
- An AI approval does not count as an approval.
- Whoever uses AI must read and understand the AI-written code before creating a PR, and is responsible for it as if they wrote it.

---

### 01 Receive request

- Who: assignee. Output: Redmine ticket.
- Every request must have a Redmine ticket. Requests received in chat or meetings: create the ticket first, then start work.
- The ticket must follow the **Ticket rules** above.

**Gate to step 02:** the ticket has all 6 items.

### 02 Analysis

- Who: assignee. The analysis does not need to be written in the ticket.
- Read the request, related documents and the code that will be affected. You must be able to explain:
  - Scope: what is done, what is **not** done.
  - Affected files, modules and systems.
  - Risks, e.g. data loss, impact on real devices, security.
  - Open questions and who should answer them.

**Gate to step 03:** questions that affect the approach have been answered.

### 03 Estimate

- Who: assignee. Approved by: PM. Output: *Estimated time* on the ticket.
- Split the work into small parts, each no more than 1 day.
- Enter the hours in the *Estimated time* field. The estimate covers coding, testing, writing the PR and addressing review.
- If during the work you see the estimate will be exceeded by more than 30%, tell the PM immediately. Do not wait for the deadline.

**Gate to step 04:** PM agrees with the estimate.

### 04 Code and local checks (hard requirement)

- Who: assignee. Output: own branch, tests and lint pass.
- Create a branch from the main branch, named by type and ticket number:
  - Feature → `feature/<ticket number>-<description>`
  - Bug → `fix/<ticket number>-<description>`
  - Example: `feature/1669-data-flow-diagram`
- Small commits, with the ticket number in the message. Example: `#1669 add error-case flow diagram`.
- Write or update tests for new logic.
- Do not commit secrets, passwords, or real customer IPs.
- Changes touching real devices (Modbus/BACnet writes, control commands) need PM approval first and may only be tried on the simulator.

#### Local checks before pushing

**Everything must pass 100% on your own machine before you push.** Do not push to let CI find errors for you.
If even one test or lint error remains, do not create the PR.

Run the commands for the languages in the change. Rules and config for each tool are in `LINT_RULES.md` at the repo root.

| Language / file type | Test | Lint / format |
|---|---|---|
| Ruby / Rails | `bundle exec rspec` | `bundle exec rubocop` |
| JavaScript / TypeScript | `npm test` | `npx eslint .` · `npx prettier --check .` |
| Python | `pytest` | `ruff check .` · `ruff format --check .` |
| Shell | — | `shellcheck` |
| YAML | — | `yamllint .` |
| Dockerfile | — | `hadolint Dockerfile` |
| Markdown | — | `markdownlint-cli2 "**/*.md"` |

Run all lints at once: `pre-commit run --all-files`

#### Not allowed

- **Disabling rules to get through:** `# rubocop:disable`, `// eslint-disable`, `# noqa` need a reason written right next to them and stated in the PR description.
- **Changing lint config in a feature PR:** including `.rubocop_todo.yml`. Rule changes go in a separate PR, agreed by the whole team.
- **Bypassing hooks:** do not use `git commit --no-verify` or `git push --no-verify`.
- **Skipping failing tests:** do not use `skip`, `xit`, `pending`, or delete tests to make them pass.

When using auto-fix (`rubocop -a`, `eslint --fix`, `ruff check --fix`), review the diff before committing.
Paste the command output into the Evidence section of the PR.

**Gate to step 05:**
- Tests pass 100% locally.
- Lint passes, with no rule disabled or test skipped without a reason.
- Self-check against the ticket's DoD is done.

### 05 Create PR with full evidence (hard requirement)

- Who: assignee. Missing evidence → sent back, not reviewed.
- One PR per ticket. Split PRs that are too large (more than about 400 changed lines).
- The PR description **must contain every section of the template**:

```markdown
## Ticket
#<ticket number> <title>

## Purpose
<Why this change is needed, 1–3 sentences>

## Changes
- <Change 1>
- <Change 2>

## Evidence
- [ ] Local tests pass: command run and result (paste output)
      `bundle exec rspec` → `<N examples, 0 failures>`
- [ ] Local lint passes: rubocop / eslint / ruff ... → `0 offenses` / no errors
- [ ] CI: green (CI link)
- [ ] Real run: screenshots, logs, or before / after video
- [ ] Mapping to the ticket's DoD: each DoD item → matching evidence

## Impact
<Affected modules, config, data, devices. Any DB migration?>

## Not done / remaining risks
<What was intentionally left out, what was not verified>

## How to review / re-check
<Steps for the reviewer to re-run>
```

#### What counts as full evidence

- **Reproducible:** state the command, environment, and input data. "Tested OK" is not evidence.
- **Matches the DoD:** each DoD condition has at least one piece of evidence.
- **Before and after** for UI, configuration, and performance changes.
- **State what was not verified.** Do not mark something "verified" if it only ran against a mock.

**Gate to step 06:**
- PR description has every section, with evidence for each DoD item.
- CI green.

### 06 Review: at least 2 approvals (hard requirement, humans only)

- Who: at least 2 reviewers. Respond within 1 working day.
- The PR author approving their own PR: does not count. AI approval: does not count.

Reviewers check:
- Evidence is complete and matches the DoD, with local test and lint results.
- No rule disabled or test skipped without a reason.
- Correct logic, error handling, no data loss.
- Security: secrets, permissions, customer data.
- Tests exist for new logic.
- Code is readable, nothing unnecessary.

Handling comments:
- Every comment must be answered or fixed. The comment's author marks it resolved.
- If code changes after approval, the reviewer re-checks that change and approves again.

**Gate to step 07:**
- At least 2 approvals, no open Request changes.
- All comments resolved.

### 07 Merge (humans only)

- Who: assignee. Output: code on the main branch.
- Merge only when **all** conditions are met:
  - At least 2 approvals, no open Request changes.
  - CI green.
  - All comments resolved.
  - Branch is up to date with the main branch, no conflicts.
- After merging: delete the branch, add the PR link to the Redmine ticket.

### 08 Deploy (humans only)

- Who: assignee. Production: PM approves first. Output: new version running on production.
- Mandatory order: **staging → production**. Never deploy straight to production, never deploy from a personal branch.

Before deploying:
- Use only code merged into the main branch with CI green.
- Prepare: version (tag or commit), config changes, environment variables, DB migrations.
- **Rollback plan:** which version to return to, whether the migration is reversible.
- Production: PM agrees first, recorded in the ticket. Tell the team the deploy time.
- If it affects edge at a site or real devices: agree on the timing with the site operator, avoid peak hours.

Staging:
- Deploy, run migrations if any.
- Smoke test main features and the changed part.
- Re-check against the ticket's DoD.

Production:
- Deploy exactly the version verified on staging.
- Smoke test right after deploying.
- Monitor logs, alerts, dashboards for at least 30 minutes.

**If an error affects users or data: roll back first, investigate later**, then tell the PM.

After deploying:
- Update the Redmine ticket: deployed version, time, smoke test result, actual time spent, status.
- Tell the requester if the ticket has met its DoD.

**Done:**
- Smoke tests on staging and production pass.
- 30 minutes of monitoring with no errors.
- Ticket updated with version, results, status.

### Setup so the system enforces the workflow

Configure branch protection for the main branch (option names as in GitHub):
- *Require a pull request before merging*, *Required approvals* = 2.
- *Dismiss stale approvals when new commits are pushed*.
- *Require status checks to pass* (CI).
- *Require conversation resolution before merging*.
- Put the PR description template in `.github/pull_request_template.md` so new PRs start with the sections filled in.
