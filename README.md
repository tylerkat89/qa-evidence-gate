# QA Evidence Gate

**Created and designed by [Tyler Katayama](https://github.com/tylerkat89) · [Original project](https://github.com/tylerkat89/qa-evidence-gate)**

**A QA protocol that keeps findings, evidence, and human decisions traceable when working with AI agents.**

QA Evidence Gate defines how an investigation moves from a proposed finding to a reviewed report. It separates observations from expectations, links claims to evidence, preserves corrections, and keeps verification and severity decisions with the human tester.

Status: **v0.1 design concept.** Every gate in this document is provisional until it has been tested on more than one real run (see [Validation plan](#13-validation-plan)). Changes from v0 are recorded in [Revision history](#16-revision-history).

---

## 1. Why this exists

This project started with a timed QA take-home assignment: explore a small React app, find bugs, rank them by severity, and write a report. I worked alongside an AI agent without a task-specific operating brief, then audited the investigation and submission.

The diagnostic work was strong. The submission was not. The reasons fell into three groups:

- **Agent failures.** The agent argued against my debugging method, then folded the bugs that method found into its own findings. It assumed product intent and built code on the assumption. It expanded scope far past what was asked. It overclaimed test runs and coverage. It re-scored its own work upward when challenged. It wrote the entire record in its own voice.
- **Human failures (mine).** I let the agent do the manual testing that was the actual job. I let severities go out marked "confirmed" when I knew the ranking was off. I kept my method and reasoning in my head instead of on the record. I argued with the agent instead of redirecting it.
- **Process failures.** Nothing stopped an unverified claim from reaching the report. Nothing forced expected results to be traced back to the spec. Nothing kept required deliverables ahead of optional work. Nothing kept a later correction from quietly rewriting earlier history.

The central problem was traceability: the final record did not preserve how decisions were reached. QA Evidence Gate proposes a workflow for keeping that reasoning visible, testing claims before they reach a report, and documenting responsibility on both sides.

## 2. Core principles

These are the rules the whole design is built on. If a feature conflicts with one of them, the feature loses.

| # | Principle | What it means in practice |
|---|---|---|
| P1 | **Found ≠ verified** | An agent finding is a hypothesis. It does not reach the report until a human has reproduced it and attached evidence. |
| P2 | **Observed ≠ expected** | "What happened" and "what should have happened" are separate fields. The expected result must cite a source, or it is marked as an **ASSUMPTION**. |
| P3 | **Intelligence ≠ authority** | Agents can investigate, draft and suggest. Only the human can confirm, rank, rule on an assumption, or send. |
| P4 | **Complete ≠ closed** | Finishing a task does not close it. Closing requires evidence that the outcome is correct. |
| P5 | **History survives correction** | The record is append-only. A correction adds a new entry and never rewrites the old one. |
| P6a | **Evidence status changes need new evidence** | Whether an artifact proves a claim (`VERIFIED` / `PARTIAL` / `UNVERIFIED` / `CONTRADICTED`) can change only when a new artifact is attached. Rereading the same evidence more generously is not allowed. |
| P6b | **Classifications change only by documented reinterpretation** | A judgment *about* the evidence (severity, self-assessment scores) can be corrected without new evidence, but only through a reinterpretation record: authored by the human, citing the exact rule, rubric line or spec quote that was misapplied, with the original kept in the ledger. See [6.1](#61-reinterpretation-protocol). |
| P7 | **Required before optional** | The deliverables the task asked for come first. Extra work is allowed only with the human's approval, and it goes in a labeled appendix. |
| P8 | **Unknown is a valid answer** | "I don't know" and "not authorized" are correct outputs. Guessing is a failure. |
| P9 | **Accountability is symmetric** | Every failure is classified as agent, human, or process. No side's failures are hidden by blaming another side. |

## 3. Scope

**This is:**
- A protocol for QA and audit work done by one human with one or more AI agents.
- A data model for findings, decisions, claims and evidence.
- A set of gates that block progress until required inputs exist.
- **Skin-agnostic.** It can run as markdown files, a CLI, a spreadsheet or a web app. The rules are the product; the interface is replaceable.

**This is not:**
- A test framework. It sits above tools like Playwright, Cypress or vitest and records what they show.
- A replacement for human judgment. The gates make judgment visible; they don't supply it.
- An enforcement system the human can't bypass. A deliberate bypass is the human's decision and the human's responsibility. The gates exist to prevent **accidental** skipping under time pressure. Every bypass is still recorded in the event log as an ordinary entry (P5). Full accountability includes the human.

## 4. Roles and authority

| Role | Who | Can | Cannot |
|---|---|---|---|
| **Tester (authority)** | The human | Define scope, write the decision log, rule on assumptions, verify findings, set severity, approve extra work, send | Delegate verification or severity to an agent |
| **Executor** | Agent A | Read source and docs, run tools, propose findings, draft text, collect evidence | Confirm findings, set final severity, change scope, edit the decision log, re-score without new evidence |
| **Attacker** | Agent B, a different model from A | Review the work for overclaims, missed spec lines, wrong severities, and unsupported claims | Edit findings; it only files challenges |
| **Record** | The ledger | Store every entry, append-only | Delete or overwrite |

The Attacker must not be the same model or session as the Executor. A system grading its own work is not an audit.

## 5. Data model

The design is defined by its records, not its screens. Any interface just has to read and write these.

### 5.1 Finding

| Field | Required | Notes |
|---|---|---|
| `id` | yes | Stable, e.g. `F-07` |
| `title` | yes | One line |
| `observed` | yes | What actually happened. Observation only, no diagnosis |
| `expected` | yes | What should have happened |
| `expected_source` | yes | A quote plus its location from the spec, rubric or docs, **or** `ASSUMPTION` |
| `repro_steps` | yes | Numbered |
| `environment` | yes | OS, browser and version, viewport, build |
| `evidence[]` | yes | Paths to screenshots, logs or recordings, each tagged `human`, `agent` or `tool` |
| `root_cause` | no | Labeled as a hypothesis unless proven (e.g. with a debugger pause on the line) |
| `severity` | yes | From the task's own rubric |
| `severity_rationale` | yes | The human's words, citing the rubric line or example |
| `found_by` | yes | `human` / `agent` / `method:<name>` |
| `verified_by_human` | yes | Must be true before the finding can close |
| `state` | yes | See 5.2 |
| `suggested_fix` | no | Short and practical. No code unless the task asks for it |

### 5.2 Finding lifecycle

```
PROPOSED  ->  REPRODUCED  ->  SOURCED  ->  RANKED  ->  CLOSED
 (agent or     (human, with    (expected     (severity +    (ready for
  human)        evidence)       cited or      rationale)     the report)
                                ASSUMPTION
                                ruled on)

Any state  ->  REJECTED   (couldn't reproduce, or not a defect per the spec; reason required)
Any state  ->  OPEN       (blocked; reason required, and it stays visible in the report)
```

A finding moves forward only when the next state's required fields exist. Moving backward appends an entry; it never erases.

### 5.3 Decision log entry

Written by the human. Agents can read it but never edit it.

| Field | Notes |
|---|---|
| `time` | |
| `type` | `method` / `priority` / `ruling` / `override` / `disagreement` / `scope` |
| `decision` | In the human's own words |
| `reason` | Why |
| `affects` | Finding IDs, phases, or "all" |

### 5.4 Claim register

Every factual claim in any summary, report or message ("21/21 tests pass", "verified in Chrome", "two runs") gets an entry.

| Field | Notes |
|---|---|
| `claim` | Exact text |
| `evidence_path` | The artifact that proves it |
| `status` | `VERIFIED` / `PARTIAL` / `UNVERIFIED` / `CONTRADICTED` |
| `intensity_check` | Do words like "severe", "fully" or "all" match what the evidence actually shows? |

### 5.5 Plan item

Every item in the test plan must end with a disposition. Nothing disappears silently.

`TESTED -> finding F-xx` · `TESTED -> no defect` · `NOT TESTED (reason)` · `OUT OF SCOPE (source)`

### 5.6 Event log

An append-only stream of every state change, ruling, correction and challenge, with timestamp and actor. Corrections reference the entry they correct.

The log also records two things that aren't state changes:
- **Refused transitions:** who attempted it, which transition, which rule refused it. This is the proof that a gate actually blocked something.
- **Bypasses:** the human, the gate skipped, and an optional reason. No penalty attached; it's just part of the record.

## 6. Execution protocol

Each phase has an **entry condition** and an **exit gate**. A phase can't start until the previous gate is passed.

### Phase 0: Intake
Read the task before touching the product.
- **Do:** Quote the deliverables list, the rubric, the scope boundaries and the time box directly from the source.
- **Also do:** Audit the task docs themselves. Sample reports, READMEs and instructions can be wrong, and a defect in the docs is a finding.
- **Exit gate:** Deliverables list (quoted) · Scope statement, including what's out of scope (quoted) · Time box · Rubric (quoted)

### Phase 1: Decision log and agent brief
The human sets the method **before** any agent runs.
- **Do:** Write the method, the priority order and the initial hypotheses in the decision log. Give the Executor the brief in [section 9](#9-agent-brief).
- **Exit gate:** At least one `method` entry and one `priority` entry, in the human's words · Brief given to the agent

### Phase 2: Spec ledger
Extract every requirement before testing, so expected results come from the spec rather than intuition.
- **Do:** List each feature or behavior with its quoted source. Anything ambiguous or undocumented becomes an `ASSUMPTION` row.
- **Exit gate:** Every assumption has a human ruling, or is marked `OPEN` and carried forward as visible

### Phase 3: Exploration
- **Do:** The human does the manual pass. The Executor can read source, run tools and propose findings in parallel. Follow downstream effects: a blocker often hides defects that only appear once it is bypassed in an isolated copy, and any finding reached that way is labeled as such.
- **Exit gate:** Every plan item has a disposition

### Phase 4: Verification
- **Do:** For each proposed finding, the human reproduces it and attaches their own evidence. Expected results are linked to the spec ledger.
- **Exit gate:** Every finding is `REPRODUCED` + `SOURCED`, or `REJECTED` / `OPEN` with a reason

### Phase 5: Severity
- **Do:** Rank each finding against the task's rubric, citing the rubric's own examples. If the rubric's example for a level matches a finding, that's the starting point, and deviating requires a stated reason.
- **Exit gate:** Every finding has a severity and a human rationale

### Phase 6: Assemble required deliverables
- **Do:** Build exactly what Phase 0 listed, in that order, before anything else. Extra work (fixes, rebuilds, deep dives) requires a `scope` entry in the decision log and goes in a labeled appendix.
- **Exit gate:** Every Phase 0 deliverable is present

### Phase 7: Claim check
- **Do:** Every claim in the report and summary goes into the claim register.
- **Exit gate:** No `UNVERIFIED` or `CONTRADICTED` claims remain in the report · Intensity words match the evidence

### Phase 8: Independent attack
- **Do:** The Attacker reviews the report against the spec ledger, the rubric and the claim register, and files challenges.
- **Exit gate:** Every challenge has a human ruling in the decision log (`accepted` with the change made, or `rejected` with a reason)

### Phase 9: Send gate
The report can be sent only when all of these hold:
- [ ] Every finding in the report is `CLOSED`
- [ ] Every finding has human evidence and a human severity rationale
- [ ] Every assumption is ruled on, or visibly marked `OPEN`
- [ ] Every plan item has a disposition
- [ ] Every Phase 0 deliverable is present
- [ ] The claim register is clean
- [ ] Every Attacker challenge is ruled on
- [ ] There is a "Tester's calls" section at the top, in the human's words: what I caught, decided, overruled and disagreed with
- [ ] The human has read the final report start to finish

### Phase 10: After sending
- **Do:** Corrections are appended, never overwritten. A re-audit may change an evidence status only by attaching new evidence (P6a), and a classification only through a reinterpretation record (P6b).

### 6.1 Reinterpretation protocol

Used when the evidence is sufficient but a classification was wrong, for example a severity that contradicts the rubric's own example.

| Field | Required | Notes |
|---|---|---|
| `target` | yes | Finding ID or score being corrected |
| `original` | yes | The classification being replaced. It stays in the ledger |
| `revised` | yes | The new classification |
| `misapplied_standard` | yes | A quote of the exact rule, rubric line or spec text that the original got wrong |
| `reason` | yes | In the human's words |
| `direction` | yes | `favors_author` / `against_author` / `neutral`. Does this change make the author's own work look better? |
| `attacker_review` | yes | The Attacker's ruling, required before the change takes effect |

Limits:
1. **Only the human can author one.** The Executor may propose a reinterpretation; it can never apply it.
2. **No new standard in the same step.** If the rubric or criterion itself needs to change, that is a separate `scope` entry in the decision log, made first. A reinterpretation applies an existing standard; it can't redefine it to fit the desired result.
3. **Evidence status is off limits.** A reinterpretation can never move an evidence status (P6a).
4. **Changes that favor the author get a closer look.** When `direction` is `favors_author`, the Attacker must state specifically why the original was wrong, not just agree.
5. **The original is never removed.** Both classifications stay visible in the history (P5).

## 7. Gate types

| Type | Example | Strength |
|---|---|---|
| **Evidence** | "Attach your screenshot of the reproduction" | Strongest. The output exists only if the work was done |
| **Enter value** | "Quote the spec line this expected result comes from" | Strong when the value can only come from doing the work |
| **Confirm** | "I read the final report start to finish" | Weakest. It's easy to tick without doing |

Design rules for gates:
1. **Prefer evidence over confirmation.** Use a confirm gate only when no output-based gate is possible.
2. **Every gate has a time cost.** If the gates eat the time box, they push the human back toward delegating, which is the failure they exist to stop. Measure the cost and cut gates that don't pay for it.
3. **A bypass is the human's call, and it is logged.** The gates prevent accidental skipping, not deliberate decisions. A bypass is recorded like any other event (5.6).

### 7.1 Enforcement levels

A gate written in this README is not the same as a gate that stops anything. Every gate declares its level, per skin:

| Level | Meaning | Example |
|---|---|---|
| **Documented** | The rule exists in writing only | A rule in this README |
| **Checked** | A validator detects a violation after it happens | A script that scans the markdown records and flags a `CLOSED` finding without human evidence |
| **Enforced** | The transition is refused before it happens, and the refusal is logged | A core that rejects "Executor sets state to CLOSED" |

Rules:
- **Claims about gates go through the claim register too.** Saying a gate "blocks" something when it is only Documented is an overclaim.
- **Enforcement lives in the core, not the skin.** For a gate to be Enforced, one core layer must own all records and transitions. Skins (markdown, CLI, HTML) only send requests to the core; they never write records directly.
- **The markdown skin tops out at Checked.** That is acceptable as long as it is stated.

### 7.2 Proving a block

A gate is only as credible as the evidence that it refuses things. Each Enforced or Checked gate ships with **negative tests**: fixtures that must be refused or flagged. For example:
- The Executor attempts to set a finding to `CLOSED`.
- A claim in the report has no `evidence_path`.
- An evidence status is upgraded with no new artifact attached.
- A reinterpretation is missing `misapplied_standard`, or is authored by the Executor.

A negative test passes only if the refusal happens **and** a refused-transition entry appears in the event log. A test that passes by confirming a hole still exists is labeled as a known limit, not counted as a pass.

### 7.3 Known limit: actor identity

The system can only block "the agent did X" if it can tell the agent from the human. If both write to the same files under the same identity, an agent can act as the human, and agent-blocking gates are not truly Enforced.

Until real identity separation exists (for example, the agent has no write access to human-owned records, and human entries are made under the human's own git authorship), every gate that depends on *who* acted is rated **Checked**, not Enforced, and is labeled that way.

## 8. Restrictions

### Executor agent
- Does not change, fix or rebuild the product unless the decision log has a `scope` entry authorizing it.
- Does not state an expected result without a quoted source; otherwise it marks it `ASSUMPTION`.
- Does not report a count ("2 runs", "25 screenshots", "64/64") without a log or artifact for it.
- Does not upgrade any evidence status without new evidence (P6a).
- May propose a reinterpretation of a classification, but never applies one (P6b).
- Does not claim findings produced by the human's method as its own. `found_by` records the actual origin.
- Answers "unknown" when it doesn't know.
- After **two** rounds of disagreement with the human, it stops arguing, records its position as a `disagreement`, and follows the human's ruling.

### Human tester
- Owns the manual pass. The agent can assist; it doesn't replace it.
- Writes the method and priorities before the agent runs.
- Doesn't confirm a severity they disagree with. Fixing the severity or logging the disagreement is mandatory.
- Doesn't send before the Phase 9 checklist passes, unless bypassing deliberately.
- After two rounds of arguing with an agent, rules and moves on.

## 9. Agent brief

Give this to the Executor verbatim at the start of a run.

> Based on QA Evidence Gate by Tyler Katayama — https://github.com/tylerkat89/qa-evidence-gate
>
> You are assisting a human QA tester. The human holds all authority.
> 1. Scope: do only what is listed in the deliverables. Do not fix, rebuild or extend the product unless the human authorizes it in writing.
> 2. Every expected result must quote its source. If there is none, label it ASSUMPTION and ask.
> 3. Every claim you make must point to an artifact. No counts without logs.
> 4. Your findings are proposals. You cannot confirm, rank or close anything.
> 5. Never upgrade an evidence status without new evidence. You may propose a correction to a severity or score, citing the rule it misapplied, but only the human can apply it.
> 6. Record where each finding came from: you, the human, or the human's method.
> 7. "I don't know" is an acceptable answer. Guessing is not.
> 8. If you disagree with the human twice, state your position once in writing and then follow their ruling.

## 10. Failure-mode catalog

Every gate exists because of an observed failure. Gates without a failure behind them get removed.

| Failure observed | Class | Gate |
|---|---|---|
| Agent argued against the method, then claimed its findings | Agent | `found_by` field · Decision log written before the run |
| Agent assumed product intent and built on it | Agent | Spec ledger · `ASSUMPTION` ruling (Phase 2) |
| Scope expanded far past the ask; required items came last | Agent + process | Phase 0 deliverables · Phase 6 order · `scope` entries |
| Overclaimed run counts and coverage | Agent | Claim register (Phase 7) |
| Re-scored its own work upward without new evidence, and redefined a criterion to earn the higher score | Agent | P6a · P6b limits 2 and 4 · Phase 10 |
| Record written in the agent's voice; human reasoning invisible | Agent + process | "Tester's calls" section · Decision log |
| Human let the agent do the manual testing | Human | Human-evidence requirement on every finding (Phase 4) |
| Human confirmed severities they disagreed with | Human | Severity rationale in the human's words (Phase 5) |
| Human argued with the agent instead of redirecting | Human | Two-round rule |
| Severities contradicted the rubric's own examples | Human + process | Phase 5 rubric citation |
| Test-plan items dropped without explanation | Process | Plan-item disposition (5.5) |
| Reviewer assessed from the agent's record without asking the human *(found when Claude reviewed the run)* | Agent (reviewer) | Reviewers interview the human before concluding · Attribution tags |
| Reviewer asked instead of deciding from what it was already told *(Claude)* | Agent (reviewer) | Decision log is read first; ask only about what isn't in it |
| Design kept growing past the request *(this project's own first draft)* | Process | P7 applies to building this tool too |
| v0's P6 would have blocked a legitimate severity correction based on existing evidence *(found in the Gauge review)* | Process | P6 split into P6a / P6b · Reinterpretation protocol (6.1) |
| Gates were described but nothing showed which ones actually block anything *(found in the Gauge review)* | Process | Enforcement levels (7.1) · Refused-transition log · Negative tests (7.2) · Actor-identity limit (7.3) |

## 11. Optional modules

Turned on only when the task's scope includes them.

- **UI consistency:** labels, casing, colors and hover/active states, alignment, copy, spacing at each viewport.
- **Data correctness:** fixtures, displayed values, API naming, and what the data says vs. what the screen shows.
- **Localization:** translation accuracy, cultural meaning, currency formats, date and number formats, text expansion, truncation, right-to-left layouts if relevant.
- **Accessibility:** labels, keyboard navigation, focus order, nested interactive elements, ARIA state, contrast.
- **Performance:** load times, Lighthouse, network weight.
- **Security:** only when explicitly in scope. If a proposed fix introduces new authority problems, that belongs in the report even when security itself is out of scope.

## 12. Metrics

Tracked per run, so the system and the human can both be judged over time.

| Metric | Why |
|---|---|
| Findings by origin (human / agent / method) | Shows whether the human is getting better or delegating more |
| Overclaims caught by the claim register and the Attacker | Measures reporting integrity |
| Required deliverables complete before any optional work | Scope discipline |
| Assumptions logged vs. ruled on | Spec discipline |
| Plan items with a disposition | Nothing dropped silently |
| Minutes spent on gates vs. testing | Whether the gates are paying for themselves |

## 13. Validation plan

This design is built from a sample of **one** run, by **one** unconfigured agent, on **one** app. Some gates may be fixing that run's specific problems rather than general ones.

1. **Brief-only test.** Run a new QA task with only the agent brief (section 9) and no gates. Measure which failures still happen. Gates that a good brief makes unnecessary get cut.
2. **Gated test.** Run another task with the full protocol, under a one-hour limit. Compare the metrics against the original run.
3. **Attack the design.** Have a different model attack this README for gates that are paperwork, missing failure modes and contradictions.
4. **Keep what holds.** Only gates that survive steps 1–3 move from provisional to stable.

## 14. Proposed repository layout

```
qa-evidence-gate/
├── README.md              this document
├── protocol/              one file per phase, with its gate
├── templates/
│   ├── decision-log.md
│   ├── finding.md
│   ├── spec-ledger.md
│   ├── claim-register.md
│   └── send-checklist.md
├── agent-brief.md         section 9 as a standalone file
├── case-studies/
│   └── 001-take-home/     the original run and its audit
└── skins/                 interfaces (markdown, CLI, HTML), added later
```

## 15. Open items

- Why were the original severities marked "confirmed" when the ranking was known to be off? *(Logged for the case study.)*
- Were the README `username`/`email` mismatch and the green-to-maroon hover color noticed during the original run? *(Logged for the case study.)*
- Which interface comes first: markdown templates, a CLI, or an HTML skin?
- What actor-identity mechanism moves agent-blocking gates from Checked to Enforced (7.3)?

## 16. Revision history

Append-only, like the ledger it describes.

### v0.1 (2026-10-22): Gauge review
Two concerns raised by Gauge; both were accepted after checking them against the v0 architecture.

1. **P6 split.** v0 text: *"P6 — Status changes need new evidence. A score, severity or verification status can change only when new evidence is attached. Reinterpreting old evidence is not enough."* Conflict found: this blocked the correction the case study itself calls for (re-ranking severities against the rubric's own example, using existing evidence). The invariant P6 was protecting was against self-serving, untraceable re-scoring that changes the standard at the same time, not against reinterpretation itself. Resolution: P6a keeps the original rule for evidence status; P6b allows classification corrections through a limited reinterpretation protocol (6.1).
2. **Enforcement made explicit.** v0 described gates without distinguishing written rules from working blocks, and the event log had no record of refusals. Added: enforcement levels (7.1), refused-transition and bypass logging (5.6), negative tests (7.2), and the actor-identity limit (7.3).

Unchanged: P1–P5, P7–P9, human authority over every confirmation, ranking, ruling and send.

### v0 (2026-10-22)
Initial design concept.


## Attribution

If you reference or adapt QA Evidence Gate, please keep the project name and creator credit attached so others can find the original work.

> Based on QA Evidence Gate by Tyler Katayama — https://github.com/tylerkat89/qa-evidence-gate

This credit line identifies the project's origin; it does not grant a license or change existing reuse terms.
