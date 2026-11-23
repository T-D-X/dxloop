---
name: dxloop
description: Structured workflow for building a feature or fixing a bug in an existing codebase: definition of done, context gathering, repro, clarifying questions, plan approval, change, proof. Use for any feature, bug, or fix request in a repo.
---

# dxloop

A disciplined loop for changing an existing codebase, whether that's a new feature, a bug fix, or a behaviour change. The point of the loop is that the code gets written *after* we agree on what "done" means and how we'll get there. Rework from a misunderstood requirement costs far more than one extra round of questions.

The user is an experienced engineer. Be direct and concise, and show your reasoning and trade-offs rather than hiding them. Prefer the simplest change that meets the definition of done over a more general or "production-hardened" design that nobody asked for.

## The loop at a glance

| # | Step | Ends with |
|---|------|-----------|
| 1 | Definition of done + brain dump | A testable DoD (drafted by you if the user didn't give one) |
| 2 | Context gathering | A short note on what you read and what matters |
| 3 | Reproduce (bugs only) | A failing test or exact repro steps, or a best-guess hypothesis if it won't reproduce |
| 4 | Clarifying questions | **Checkpoint: wait for answers** |
| 5 | Own words | Your restatement of the problem and constraints |
| 6 | Plan + trade-offs | **Checkpoint: wait for approval** |
| 7 | Make changes | Small, reviewable increments |
| 8 | Verify and prove | Evidence mapped to each DoD item |

There are two hard stops, at steps 4 and 6. Don't write production code before the plan is approved. The only exception is when the user explicitly says to skip ahead ("just do it", "no need to check with me"). Exploratory code to reproduce a bug or test an assumption is fine before approval. Say that it's throwaway, and don't leave it in place.

Steps 2, 7 and 8 can hand some work to subagents once a task passes a certain size. See **Delegating to subagents** near the end.

---

## 1. Definition of done + brain dump

The user will usually open with a brain dump that may be messy, partial, or mixed with implementation ideas. Your job is to pull out a **definition of done**: a short list of observable, testable outcomes.

- If the user gave a DoD, use it as written.
- If they didn't, draft one from the brain dump and mark it as a draft so they confirm it at step 4.
- Each item should be checkable: "POST /webhooks returns 200 and enqueues exactly one job for a duplicate message ID", not "webhooks work better".
- Separate **in scope** from **out of scope** when the brain dump hints at adjacent work. Scope creep usually starts with an unspoken "while I'm in here".
- Classify the request as **feature**, **bug**, or **both**. The classification decides whether step 3 runs.

## 2. Context gathering

Read before reasoning. In roughly this order, and skip whatever doesn't exist:

1. **Rules and standards**: `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, lint and formatter configs, and any coding-standards doc. These override your defaults.
2. **Project docs**: `README`, `docs/`, architecture notes, and **ADRs** (`docs/adr/`, `adr/`, `decisions/`). ADRs tell you why things are the way they are, so don't propose undoing an accepted decision without flagging it.
3. **Related code**: the entry point for the behaviour, its callers and callees, and the data models involved. Follow the actual call path instead of guessing from file names.
4. **Existing tests** for that area: how they're structured, what fixtures exist, and how to run them. You'll need this for steps 3 and 8.
5. **History**: `git log` and `git blame` on the key files, when the reason behind the current code matters, especially for bugs.
6. **Relevant skills**: if other available skills cover part of this work (a framework, a file format, a deployment target), load them now.

Find out how to run the tests, linters, and type checks (check the Makefile, `package.json` scripts, `pyproject.toml`, CI config, and so on) and note the exact commands.

Always read the rules and standards (item 1) yourself, because they govern every later step. If the rest of the search is broad, hand it to a read-only search subagent that returns conclusions and file paths. Broad means the area is unfamiliar, spans several modules, or would take more than about ten files to map. See **Delegating to subagents**.

End with a brief note: what you read, the files that matter, and anything surprising. Keep it to a few lines, because a file-by-file tour isn't useful.

## 3. Reproduce the bug (bugs only)

A bug you can't reproduce is a bug you can't prove you fixed, so make a real effort here.

- Best case: write a **failing automated test** that captures the bug at the lowest sensible level. The test becomes the regression test in step 8.
- If a test isn't practical, give exact manual repro steps (command, input, observed output vs expected output).
- Record the actual error or output verbatim.
- Once it's reproduced, form a root-cause hypothesis and point to the line or condition responsible. Treat "the symptom goes away if I change X" as something to investigate, not as a root cause.

**If you can't reproduce it**, don't stall. Make an educated guess about the likely cause from the brain dump and what the code shows: the most plausible code path, condition, data shape, or environment difference that would produce the symptoms described. Then take the guess into step 4 to confirm:

- State the hypothesis concretely ("I think `dedupe_key` is None for messages without a `context` field, so the uniqueness check is skipped") and say how confident you are.
- Say briefly what you tried in order to reproduce it.
- Ask for whatever would confirm or refute it: logs, a sample payload, environment or version details, or whether the hypothesis matches what they've seen.
- If the hypothesis stands, write a test that encodes the suspected failure condition, so there's still a before/after even though the original report wasn't reproduced.

## 4. Clarifying questions — CHECKPOINT

Ask only the questions whose answers would change what you build. Good questions resolve ambiguity in the DoD, pick between real design options, or confirm assumptions with consequences, such as data migrations, API contracts, or backwards compatibility. Bad questions ask about things you could have found in the code.

- Number them so the user can answer by number.
- For each one, give your default assumption: "1. Should duplicates be dropped silently or logged? *Default: log at INFO.*" The user can then reply "defaults fine except 2".
- If you drafted the DoD, include it here for confirmation.
- If the bug didn't reproduce, put your hypothesis first (see step 3), because the rest of the plan depends on it.
- If there's genuinely nothing to ask, say so and state the assumptions you're making. Then continue to step 5 in the same message.

Then **stop and wait**.

## 5. Own words

After the answers come in, restate your understanding in your own words. This catches misunderstandings before they turn into code. Cover:

- **The problem**: what's wrong or missing, and for bugs the root cause. Label it as confirmed or still a hypothesis.
- **The goal**: the confirmed DoD.
- **Constraints**: standards, ADRs, compatibility, and anything the user said in step 4.
- **What won't change**: explicit non-goals.

Keep it short and avoid parroting. If the restatement sounds identical to what the user wrote, it isn't proving you understood.

## 6. Plan + trade-offs — CHECKPOINT

Send this in the same message as step 5, directly after the restatement, so the user can correct the understanding and the plan together.

- **Approach**: the files and functions to change and what changes in each, in order. Keep each step small enough to review on its own.
- **Options considered**: if there was a real choice, give 2–3 options with trade-offs (complexity, risk, performance, consistency with existing patterns, reversibility) and say which one you recommend and why. When the choice is obvious, say so in one line instead of inventing alternatives.
- **Tests**: which tests you'll add or change, and what each proves against the DoD.
- **Risks**: migrations, behaviour changes for other callers, and anything hard to roll back. If the root cause is still a hypothesis, list that as a risk too.
- **Docs**: whether docs need updating, and whether this decision deserves an ADR. Add one if it's an architectural choice someone will later wonder about.
- **Delegation**: if you plan to use subagents in step 7 or 8, say so in one line: which chunks or checks, and why.

Then **stop and wait for approval**. If the user changes the plan, acknowledge the change in a line and go ahead. You don't need to re-present the whole plan unless it changed substantially.

## 7. Make changes

- Follow the approved plan. If reality diverges, such as a hidden caller or an assumption that turns out wrong, **pause and say so** before improvising a different design. A small, obviously necessary adjustment can just be noted. A change in approach needs a check-in.
- Match the codebase's existing patterns, naming, and style, even where you'd personally do it differently.
- Work in increments: change, run the relevant tests, move on. Don't batch everything and test once at the end.
- No drive-by refactors or unrelated cleanups. If you spot something worth fixing, list it as a follow-up at the end.
- Remove any throwaway code from steps 3 and 6.
- By default, make the changes yourself. Parallel subagents only make sense when the approved plan has independent chunks. See **Delegating to subagents**.

## 8. Verify and prove

"It should work" doesn't count as verification. Show evidence:

1. Run the **new and changed tests** and show the command and result.
2. For bugs: show the repro test **failing before** the fix and **passing after**. If you wrote it at step 3, you already have the "before". If the original bug never reproduced, say so plainly: the test proves the hypothesised cause is fixed, and confirming the real-world symptom is gone is still up to the user.
3. Run the **wider suite**, or at least the affected module's tests, plus lint, type checks, and formatting as the repo defines them. Report anything that fails, including pre-existing failures, and say clearly which failures are pre-existing.
4. Where tests can't cover something (UI, an external integration, a config change), do a manual check and describe exactly what you did and saw.
5. For risky changes, get a **fresh-eyes review** from a subagent that didn't write the code (see **Delegating to subagents**). Report what it found and what you did about each point.

Finish with a **DoD checklist**: each item from the definition of done, ticked, with the evidence that proves it (test name, command output, or manual check). Any item that isn't fully met gets flagged plainly with the reason.

Then close with:
- A one-paragraph summary of the change (suitable as the basis for a PR description).
- Files changed.
- Follow-ups you noticed but deliberately didn't do.

---

## Delegating to subagents

A subagent starts cold. It has none of this conversation, so it has to rebuild context from your brief and its own reading. That makes delegation **more** expensive in total for small tasks. What it buys is a clean main context: noisy output such as file dumps, long test logs, and search results stays inside the subagent, which hands back only a conclusion. On a long task that saving compounds, because everything in the main context is carried through every later turn. A second benefit is independence: a reviewer that didn't write the code isn't grading its own work.

So delegate only above a size threshold, and only in these places:

| Where | Delegate when | What the subagent returns |
|---|---|---|
| Step 2, context search | The area is unfamiliar, spans several modules, or would take more than about ten files to map | Call paths, relevant files with one-line roles, test locations, ADRs that apply |
| Step 7, implementation | The approved plan has **independent** chunks (separate files or modules, clear interfaces, no shared in-progress state), and there are enough of them that parallel work is worth the handoff | The chunk implemented, with its tests run and results reported |
| Step 8, test runs | The full suite or lint output is long and you only need pass/fail plus the failures | Command run, pass/fail counts, failures verbatim, and which failures are pre-existing |
| Step 8, fresh-eyes review | The change is risky (auth, payments, data migrations, deletion, concurrency, public API contracts) or large | Concrete issues against the DoD and the repo's standards, each with file and line |

Never delegate steps 3–6. Reproducing, asking, restating, and planning depend on the user's answers and on keeping the reasoning in one place. Splitting them loses exactly the context those steps exist to build. Below the threshold, do everything directly. A small fix should never spawn a subagent.

**Writing the brief.** Because the subagent knows nothing, the brief has to carry the context:
- The goal and the relevant DoD items.
- Constraints: the repo's rules from `CLAUDE.md` and the standards docs, plus the parts of the approved plan that apply.
- Exact scope: which files it may touch, and that it must not touch anything else.
- What to return, and in what shape (see the table). Ask for conclusions, not raw dumps.
- For a reviewer, the diff and the DoD, but not your reasoning about why the code is right. The point is an independent read.

Leave model and effort choices to the environment's own routing, and don't specify them in the brief.

**Treat what comes back as input, not truth.** Spot-check a subagent's key claims (a file it says is the entry point, a test it says passed) before relying on them, and integrate its work yourself. You still own the final DoD checklist.

---

## Scaling the loop

A one-line fix doesn't need a two-page plan, so compress the content: a two-line restatement, a three-bullet plan, one question or none. **Keep the checkpoints even when the content is short.** The checkpoints are what make this workflow different from "just code it". If the user wants to skip them for a particular task, they'll say so.

For large features, apply the loop per chunk. Agree on the overall plan once, then run steps 7–8 for each chunk and check in between chunks instead of delivering everything at the end. Large features are also where delegation in step 7 is most likely to pay off.
