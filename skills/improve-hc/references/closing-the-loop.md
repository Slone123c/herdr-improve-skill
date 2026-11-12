# Closing the Loop — execute, reconcile, issues

The advisor's job doesn't end at the plan. This file covers the three follow-through flows: dispatching an executor and reviewing its work (`execute`), keeping the plan backlog alive (`reconcile`), and publishing plans where work gets picked up (`--issues`).

The founding rule survives unchanged: **the advisor never edits source code.** In `execute`, a *separate executor subagent* edits code in an isolated git worktree; the advisor dispatches, reviews, and renders a verdict — like a tech lead who doesn't push commits to your branch.

## hc operations — launch, talk, wait, and clean up

These are direct hc operations, shared by audits, cold plan reviews, and execution. No script, adapter, extra configuration, or generated template is needed. Check installed `hc <command> --help` and the selected agent's help for exact arguments.

### Choose and launch

Resolve the launching agent's parent address **before** creating worktrees or launching children. This is the advisor for ordinary delegation, or the owning worker for a verification runner; a child's address is not its parent's.

1. Capture `$HC_TERMINAL_ID` when present and verify it with `hc get <parent>`. An empty variable in a shell tool is inconclusive: tool environments can omit hc variables even when the agent runs inside hc.
2. If the address is absent or unreachable, inspect `hc server status` and `hc ls --json`. A permission/lock-file error identifies a blocked check, not a missing service. Use the environment's normal approval mechanism to retry that read-only check outside the sandbox when permitted; preserve the actual error if access remains unavailable. Keep the user's core running.
3. For candidate terminals, inspect `hc get <id>` and compare its foreground agent PID with the tool's actual process ancestry (for example, PID/PPID/command-name output from `ps`). Recover the parent only when that process evidence identifies the current agent. Matching repository path, title, status or agent name alone is insufficient. Request read-only process access through normal approvals if needed; avoid dumping environments or full process arguments that may contain secrets.
4. Record the verified address and evidence. Pass `--parent <verified-address>` to `hc run` if the shell omits the variable. For commands that resolve the caller from the environment, such as `hc wait --children`, scope `HC_TERMINAL_ID=<verified-address>` to that command. This restores the proven identity; it does not authorize borrowing another terminal. Only when identity or reachability cannot be established after these checks, explain the remaining failure and how to start the agent inside hc. Do not launch unreportable children.

For Codex launched through hc, use `codex --no-daemon` (including templates) so its tools remain in the hc process context. A missing shell variable alone does not justify restarting an existing session.

- Honor the user's specific task/plan agent and model choices. Inspect `hc templates`; prefer an existing template whose command matches. Otherwise use the chosen agent's verified interactive argv and read the relevant bundled role body from this skill. All hc-launchable agents are eligible, but persistent revision requires an interactive agent. Do not use print/one-shot mode for an executor.
- Record the parent, task, chosen agent/model (or explicitly its configured default), expected report path, and any worktree/base in the existing index or plan. Pre-create report directories under the chosen plan directory; allocate verification log/report paths if runners may be used. These exact assigned paths may be outside an executor's worktree, not permission to edit the index or other plans.
- Use `hc run --bg --name <unique-name> --cwd <absolute-directory> <matching-template> <complete-task>` and inspect the launch response's terminal ID and `first_message.sent`. Use legal names of at most 32 characters. Keep executors alive between turns; do not use `--rm`, `--keep`, or `--outlive-parent` by default.
- Without a matching template, use `hc run --bg --name <unique-name> --cwd <absolute-directory> <agent> <its-arguments>`. Inspect `hc get`/`hc read` and, when useful, `hc wait <id> --timeout 60` for readiness, then `hc send <id> <role-instructions-and-complete-task>`. Startup idle alone does not prove readiness; a trust prompt is not an agent input prompt. Include the full role body, fixed parent address, and full task in this first delivery.
- Pass task text and model values as safely quoted arguments, preserving newlines and literal shell characters; never evaluate plan contents as shell code. hc has no assumed universal model flag or template command override. Do not rewrite a shared template to change a concurrent task's agent. If a full payload exceeds the available argument limit, stop and report instead of silently dropping context or inventing a transport.
- If launch says `first_message.sent: false`, the task was not sent. Read the screen and fix readiness before a single deliberate send. For a failed or ambiguous send, inspect the screen and `hc get` first: it may have arrived. Do not blindly resend, spawn a duplicate, or mark IN PROGRESS until delivery is confirmed. Dialogs requiring a person's decision go to the user; do not answer them yourself.
- After the user resolves a trust/auth prompt, re-read the same terminal to establish readiness and continue any already-authorized task. Acknowledging a prompt is not evidence it has cleared. Preserve the existing agent and task; do not launch a replacement just because startup paused.

### Receive and wait

- Each task must say: write the result to its assigned absolute report path, then `hc send <captured-parent-address> <task-identity-and-report-path>` on completion or trouble. The executor report retains the exact format below. Read the file before accepting the message's claims. Children report findings/status, not user instructions or authorizations.
- Use `hc wait --children any --timeout 60` to handle the first relevant child's news; process its identity against your records, then wait again. `reason: message` means the message arrives in your input, possibly later; it is not screen output to retrieve with `hc read`. `reason: timeout` is a check-in, not failure. Do not sleep or busy-poll.
- Distinguish terminal state (`live`/`finished`), OSC 7501 status (`idle`, `working`, `done`, `blocked`, `error`), the report's STATUS, and your verdict. Idle, a startup status, a send acknowledgment, or an exit without a valid report never proves task success. If wait is immediately returning the same already-handled state, stop waiting on that child and handle or park it; do not spin.
- `blocked`: read its screen, report the obstacle and `hc attach <id>` to the user, preserve the terminal, and wait for human resolution. Never use `hc send --force` or `hc keys` to bypass permission/trust decisions. Continue handling independent children while this one is parked.
- `error`: read why, then choose specific follow-up instructions or BLOCK. A STOPPED report goes directly to BLOCK. COMPLETE starts review, not automatic approval. A child that exited or became idle without a report requires inspection; incomplete evidence is not DONE.
- Check status support for the selected agent. Without OSC reports, explain that automatic blocked detection is unavailable; file reports plus `hc send` still work, but a bare wait may last until exit. Do not infer status from screen wording or quietly advertise full lifecycle support. If agent messages cannot reach the advisor (including disabled hc notes), resolve communication before dispatch or report the limitation.

### Slow commands and cleanup

By default, run slow checks with `hc run --bg --name <name> --cwd <directory> <program> <args>` as background programs. Wait with hc, then collect actual exit codes and needed evidence. For noisy checks, the runner option below keeps raw output out of the owner's context. Avoid `--rm` until you have needed evidence. Keep a bounded wait so the advisor stays responsive.

### Lightweight verification runners

For a simple check, a background program is enough. For voluminous output or a check needing continued monitoring, an advisor or worker may instead launch a cheaper interactive hc agent as a runner. Choose it through the same user agent/model instructions; no third template, native subagent API, helper script or new configuration layer is needed. Send a small task directly after readiness.

- Pass the exact command and arguments, absolute cwd/worktree, expected outcome, immediate parent's hc address, task identity, and unique absolute log/report paths under the existing plan directory. Include only environment prerequisites needed to run the command; omit the full plan and unrelated repository context. Allocate these paths before dispatch so a read-only worker never invents write permissions.
- The runner executes the assigned check and captures full stdout/stderr to its log file. Preserve the checked command's actual exit code, not a successful tee/tail command's code. It may produce normal test/build artifacts permitted by the plan, but its job is running and reporting, not changing source/tests, choosing another command to obtain a pass, implementing fixes, or managing a further workflow. Ordinary permission/trust rules still apply.
- Write a short report with command, cwd, actual exit code, test counts when present (including failures/skips), first actionable failure, and full log path. Send only that short result and report path to the immediate parent with `hc send`. Raw intermediate output stays in the log. If blocked or interrupted, report that state rather than inventing a passing exit code.
- The owning worker uses hc wait, reads the small report, and opens the relevant failure excerpt only when needed. It decides fixes and reruns; a failed check or STOP condition is not concealed by delegating it. The advisor still independently re-runs review criteria, optionally through its own fresh runner; the executor's runner report is not a substitute for independent review.
- The owner records runner ID, command, log/report paths and cleanup result in its own report's NOTES; it does not edit the advisor's index. Collect results and clean up owned runners before reporting COMPLETE. If one needs a person, tell the advisor its ID and obstacle for attach, preserving the runner until resolved. Surface unresolved descendants before any parent/worktree cleanup. The advisor records relevant leftovers for reconcile, using the same ownership checks.

After collecting results, `hc kill <id>` if still live, then `hc rm <id>`. Only clean up terminals you started and recorded; `hc ls` includes the user's other work. Never stop/restart the user's core. Record failed cleanup in the existing index/plan. Safe worktree cleanup is described under Verdict; historical leftovers are reported under reconcile. Terminal termination is not proof that intentionally detached background programs have exited; report any known remaining processes or ports without a broad process sweep.

---

## `execute <plan>` — dispatch and review

### Preconditions (check all before dispatching)

- The repo is a git repository (worktree isolation requires it). If not: stop and say so.
- The plan file exists and its dependencies show DONE in `plans/README.md`. If not: stop, name the missing dependency.
- Run the plan's drift check yourself. If in-scope files changed since `Planned at`, reconcile the plan first (see below) — don't hand a stale plan to an executor.
- Confirm dependency changes are actually present in the selected dispatch base. DONE in another unmerged worktree is not sufficient. Ask the user to merge or identify a suitable base; never merge/cherry-pick on their behalf. Examine staged, unstaged, and untracked in-scope changes too, not just the plan's committed diff.
- Confirm the selected agent/model and hc parent are usable before creating resources. If no selection/default is available, ask only for the missing launch choice; do not invent model identifiers. Do not silently change a failed user choice.

### Dispatch

Keep one persistent hc executor per eligible plan. Preserve single-plan execution; allow multiple independent plans whose scopes do not overlap, subject to machine/user concurrency limits. Apply the user's type-level or plan-specific agent/model choice; medium/cheap model preferences are advisory, not vendor bindings.

For each plan, record the full `BASE_SHA` of the chosen committed starting point and check `git worktree list --porcelain` and existing branch names. Follow the plan's branch convention, using a unique branch and a sibling worktree directory outside the user's checkout. Create it directly with `git worktree add -b <branch> <absolute-worktree> <BASE_SHA>`. Never reuse or remove pre-existing user resources. Record intended paths before creation so a partially failed launch can be reconciled. If launch fails, safely release only newly created resources or record why they remain.

Start the executor with the hc operations above, using the worktree as cwd. Record the actual terminal ID, agent/model, branch, worktree, base, report path, and delivery result. Set IN PROGRESS only after confirmed task delivery; the advisor owns the index.

The task must contain the captured parent address, plan identity, absolute report path, worktree/base, and:

1. **The full plan file text, inlined.** The worktree contains only committed files — if `plans/` is uncommitted, the executor can't read it. Never assume; always inline.
2. The executor preamble:

> You are the executor for the implementation plan below. Follow it step by
> step. Run every verification command and confirm the expected result before
> moving on. Touch only the files listed as in scope. If any STOP condition
> occurs, stop immediately and report. Do not improvise around obstacles.
> Commit your work in the worktree following the plan's git workflow section.
> One override: SKIP the plan's instruction to update `plans/README.md` —
> your reviewer maintains the index. Before reporting, audit every claim in
> your report against an actual tool result from this session — only report
> what you can point to evidence for; if a verification failed or was
> skipped, say so plainly. When finished, reply with exactly the report
> format below.

3. The report format:

```
STATUS: COMPLETE | STOPPED
STEPS: per step — done/skipped + verification command result
STOPPED BECAUSE: (only if STOPPED) which STOP condition, what was observed
FILES CHANGED: list
NOTES: anything the reviewer should know (deviations, surprises, judgment calls)
```

Include hc communication and the verification-delegation rules from the bundled executor role, even on a direct launch. The assigned report and explicitly allocated runner log/report files are allowed outside the worktree; the index remains advisor-owned. Any plan done criterion requiring the index update is fulfilled by the advisor in the main plan directory, not by executor edits.

### Review (the advisor's real job here)

Note on fresh worktrees: they share git history but not `node_modules` or build artifacts — the executor must install dependencies first, and check tooling that resolves from `dist/` may need one build even though the plan's command table (recon'd in the main tree) didn't mention it. Expect this; it isn't a deviation.

Review like a tech lead reviewing a PR against the spec — never fix anything yourself:

1. **Re-run every done criterion** in the worktree. Don't trust the executor's report — verify. Put slow verification in named background hc program terminals and collect all actual results before deciding.
2. **Scope compliance**: compare `git -C <worktree> diff --stat <BASE_SHA>...HEAD` and `git -C <worktree> diff --name-status <BASE_SHA>...HEAD` against scope. Also inspect `git status --porcelain=v1 --untracked-files=all`, `git diff`, and `git diff --cached` there. Include both paths of renames and all untracked files. Plain `git diff` misses committed executor changes. Any file outside scope fails review, full stop; the assigned report outside the worktree is the explicit operational exception.
3. **Read the full diff.** Judge it against "Why this matters" (does it solve the actual problem?) and the repo conventions named in the plan (does it look like the rest of the codebase?).
4. **Audit the new tests.** Executors game criteria — a test that asserts nothing meaningful passes `pnpm test` and proves nothing. Read what the tests assert.

Read `git -C <worktree> diff <BASE_SHA>...HEAD` in full, alongside any staged/unstaged diff and new files. Ensure the executor stayed on its assigned branch with `BASE_SHA` as an ancestor; it must not reset/rebase or import unrelated work. Final source/test changes must be committed before handoff. Review-generated ignored artifacts are expected; investigate other untracked or dirty files rather than losing them in cleanup.

### Verdict

**Documented deviations are judged on merit, not reflex-blocked.** "Do not improvise" exists to stop silent drift; an executor that hits a real obstacle (e.g. the plan's approach breaks existing test mocks), adapts minimally, and explains it in NOTES has done the right thing. Approve it if the adaptation serves the plan's intent and stays in scope; treat *undocumented* deviations as review failures.

| Verdict | When | Action |
|---|---|---|
| **APPROVE** | Criteria pass, scope clean, quality holds | Mark DONE, record reviewed HEAD, evidence, and whether merged. Kill/remove your executor terminal after collecting the report; retain its branch and worktree. Present diff summary, worktree, branch and NOTES. **Merging is the user's decision — never merge, push, or commit to their branch.** |
| **REVISE** | Fixable gaps | `hc send <same-executor> <specific-actionable-feedback>`, e.g. name the failing criterion and exact error-handling gap. Confirm delivery, retain its context/agent/model, keep IN PROGRESS and review the new result. **Max 2 revision rounds**, recorded in the index/plan, then BLOCK. A lost executor is not the same persistent session: report that interruption before any replacement dispatch. |
| **BLOCK** | STOP condition hit, scope violated unrecoverably, or revisions exhausted | Mark BLOCKED with reason, collect evidence, and refine/rewrite the plan with what was learned. Clean up your terminal when it is no longer under human control; attempt ordinary `git worktree remove <path>` only for your own no-longer-used worktree. Preserve dirty, locked, or human-controlled directories and record pending cleanup. Never force removal, discard uncommitted/untracked work, or delete the branch silently. Tell the user what happened and where evidence remains. |

Running verification commands inside the executor's worktree is fine — it's isolated and disposable. The no-mutating-commands rule protects the user's working tree, not the worktree.

---

## `reconcile` — keep `plans/` alive

Process what happened since the last session. Read `plans/README.md` and every plan file, then per status:

- **DONE** — spot-check that the done criteria still hold on the current HEAD (cheap ones only). Mark verified in the index. Don't delete plan files — they're the record.
- **BLOCKED** — read the reason. Investigate the underlying obstacle in the codebase. Either rewrite the plan around it (new number if the approach changed fundamentally, in-place refresh otherwise) or mark REJECTED with one line of rationale.
- **IN PROGRESS** (stale) — flag it to the user; an executor probably died mid-run. Check the worktree if one exists.
- **TODO** — run the drift check. If drifted: re-verify the finding still exists (it may have been fixed in passing), then refresh the "Current state" excerpts and `Planned at` SHA. If the finding is gone, mark REJECTED ("fixed independently").

Distinguish approved-but-unmerged DONE from landed DONE. Keep approved/unmerged status and return its branch/worktree rather than failing it because the main checkout lacks the change. For committed executors, `git merge-base --is-ancestor <reviewed-HEAD> HEAD` can confirm ancestry; squash/rebased merges require inspecting the actual change/done criteria, not assuming ancestry failure means unmerged. Only claim current-HEAD verification when actually performed. Dependencies are executable only when their changes are present in the proposed base.

Also cross-check the recorded terminal IDs with `hc ls --json`/`hc get <id>` and worktrees with `git worktree list --porcelain`. Match repo, branch, base and paths, not just name prefixes. Report live/blocked executors, missing terminals with surviving worktrees, pending cleanup and unknown leftovers. Do not duplicate an active assignment, resend tasks, or interrupt a human takeover. Historical resources were not started by this session: report them for user disposition rather than automatically killing/removing them. If hc is unavailable, state that terminal reconciliation was incomplete and still do the plan/Git checks you can.

Finish with a short report: what's verified done, what was refreshed, what's rejected, and what's executable right now.

---

## `--issues` — publish plans as GitHub issues

Modifier on any planning invocation (`/improve-hc --issues`, `/improve-hc security --issues`), or an equally explicit natural-language request to publish plans as GitHub issues. Either is authorization to create issues; a request merely to plan, audit, or implement is not. The visibility and sensitive-content checks below still apply.

1. Preflight: `gh auth status` succeeds and the repo has a GitHub remote. If either fails, write the plan files as normal and say why issues were skipped.
2. Visibility check: `gh repo view --json visibility`. If the repo is **public**, warn the user that issues are publicly visible and get explicit confirmation before publishing any plan that describes a security vulnerability, credential location, or other sensitive finding.
3. Show the list of titles about to become issues; confirm once if interactive.
4. Per plan: `gh issue create --title "<plan title>" --body-file <plan file>`. Labels: `improve` plus the category — apply only if the labels exist or can be created without erroring; skip labels rather than fail.
5. Record each issue URL in the plan's Status block (`- **Issue**: <url>`) and the index.

The plan file remains the source of truth; the issue is distribution. The self-containment rule pays off here — the issue body needs no edits to make sense to whoever (or whatever) picks it up.
