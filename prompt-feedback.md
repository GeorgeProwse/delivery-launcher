# Prompt Feedback: PwC Engagement Planning Framework

**Branch:** `claude/improve-pm-prompts-cjj2g`
**Reviewed:** 2026-02-22
**Scope:** All 19 prompts (P1.1–P2.13) as currently inlined in `index.html`

---

## How to Read This Document

Each section maps to one of the 7 objectives. A cross-cutting analysis covers issues affecting multiple prompts first. Per-prompt feedback follows. A priority matrix is at the end.

**Objectives:**
1. Outputs use best-practice PM methodology
2. Outputs are professionally formatted and high quality
3. Outputs are consistently relevant to context and to each other
4. Outputs are generated without the LLM crashing
5. All outputs can be produced without hitting the maximum context window
6. LLM uses code interpreter to generate files but does not show all reasoning or coding
7. LLM uses only one code interpreter call per output

---

## Part 1 — Cross-Cutting Issues

These issues affect groups of prompts. Fix them first; they unlock the most improvement per unit of effort.

---

### Issue CX-1: Two-Tier Prompt Quality Split (HIGH PRIORITY)

The 19 prompts divide into two distinct quality tiers:

**Tier A — Refactored (clean, consistent):**
P1.1, P1.2, P1.3, P1.4, P1.5, P1.6, P2.1, P2.4, P2.7, P2.8, P2.9, P2.10, P2.11, P2.12, P2.13

**Tier B — Partially refactored (still carrying legacy issues):**
P2.2 (24,848 chars), P2.3 (23,383 chars), P2.5 (21,769 chars), P2.6 (26,477 chars)

**Differences between tiers:**

| Property | Tier A | Tier B |
|---|---|---|
| Old PwC format spec (ITC Charter / #D04A02 hex) | Removed | Still present |
| JSON canonical handoff output | Present | Absent |
| Internal step numbering | Consistent | STEP 1 → STEP 3 (STEP 2 missing) |
| Internal contradiction (clarification + no-pause) | Resolved | Still present |
| Char count | 5,527–15,774 | 21,769–26,477 |

**Recommendation:** Complete the refactoring of P2.2, P2.3, P2.5, P2.6 to match the Tier A pattern. These four prompts affect Objectives 1, 2, 3, 4, 5, and 7.

---

### Issue CX-2: {{PROMPT_DATA}} Placeholder Presence (MEDIUM)

Every Tier A prompt except P1.1 ends with the line `{{PROMPT_DATA}}`. This placeholder is undefined in the prompt text and it is unclear whether it injects anything at runtime.

- If it injects the full prior step outputs: it is a context-window risk (Objective 5).
- If it is unused: it is dead text that may confuse the LLM.
- If it is injecting engagement brief data: P1.1 already handles this via `{{ENGAGEMENT_DATA}}` and the canonical handoff. Duplicating it breaks the chain architecture.

**Recommendation:** Clarify and document what `{{PROMPT_DATA}}` injects at runtime. If it is unused, remove it from all prompts. If it injects prior-step summaries, cap injection to the canonical JSON handoff object only (not full document text).

---

### Issue CX-3: Missing JSON Handoff in P2.2, P2.3, P2.5, P2.6 (HIGH)

Tier A prompts produce a `PX_X_CANONICAL_HANDOFF` JSON object in-chat for downstream steps to consume. Tier B prompts (P2.2, P2.3, P2.5, P2.6) do not — they still rely on the older "refer to documents in this message thread" approach.

This breaks the chain architecture: downstream prompts (P2.4, P2.7, P2.8, P2.13) that expect to parse structured JSON from prior steps will find nothing from P2.2, P2.3, P2.5, or P2.6.

**Recommendation:** Add a canonical JSON handoff output contract to P2.2, P2.3, P2.5, and P2.6 using the same pattern as Tier A prompts.

---

### Issue CX-4: "Do Not Pause" Inconsistency Across Tier A (MEDIUM)

Of the 15 Tier A prompts, 9 include `do NOT pause` or equivalent no-wait instruction; 6 do not (P1.1, P2.10, P2.11, P2.12, P2.13, and partially P1.2).

- P1.1 is correct: it produces only a JSON block, no clarification needed.
- P2.10, P2.11, P2.12, P2.13: absence of the no-pause rule means the LLM may insert a confirmation gate before file generation. This increases session length and context consumption.

**Recommendation:** Add the no-pause execution instruction explicitly to P2.10, P2.11, P2.12, and P2.13 to make all Tier A prompts consistent on this point.

---

### Issue CX-5: Repeated PwC Formatting Spec in P2.2, P2.3, P2.5, P2.6 (MEDIUM)

P2.2, P2.3, P2.5, and P2.6 still embed the full PwC typography, colour palette, and table formatting specification in their body text (referencing both old hex codes AND new SS-1 tokens, creating an inconsistency). Each instance adds approximately 3,000–4,000 characters of duplicate text.

Tier A prompts handle this correctly: they reference SS-1 only and do not restate it.

**Recommendation:** Remove the old formatting spec block from P2.2, P2.3, P2.5, and P2.6. Replace with a single line: `Apply SS-1 (Word) / SS-1 (Excel) throughout. Do not restate formatting rules.`

---

### Issue CX-6: No Explicit Rule to Suppress Code Display (MEDIUM — Objective 6)

The EXECUTION CONSTRAINTS block (present in all prompts) addresses when and how many times code interpreter is called. It does not explicitly instruct the LLM to suppress display of its Python source code in the response text.

Rule 1 says "Do all parsing, validation, reasoning, and content drafting in your response text only" — this could encourage the LLM to narrate or display its reasoning at length before the code call.

**Recommendation:** Add rule 10 to the EXECUTION CONSTRAINTS block across all prompts:

> 10. Do not reproduce, display, or summarise your Python script in response text. Show only: (a) a one-line confirmation that the file has been generated, (b) the download link, and (c) your closing message. Any reasoning or schema decision-making must be internal and not output to the user.

---

### Issue CX-7: Context Window Accumulation Risk (MEDIUM — Objective 5)

Prompts run sequentially in a single chat session. By the time P2.13 runs, the context must hold:
- 18 prior prompt texts (~230k chars / ~57k tokens)
- 18 canonical JSON handoff objects (varies, but potentially 5–15k tokens total for rich objects)
- Any LLM reasoning text and closing messages per step

**Current total prompt text alone: ~265k characters / ~66k tokens.** This is before any outputs are added.

Most commercial LLM sessions (128k–200k token context windows) will be under pressure by P2.8–P2.10, and at serious risk of truncation by P2.13.

**Specific concern:** P2.2 (6,212 tokens), P2.3 (5,845 tokens), P2.5 (5,442 tokens), and P2.6 (6,619 tokens) are the four largest contributors. Trimming these four alone could recover ~10,000 tokens.

**Recommendations:**
1. Trim P2.2, P2.3, P2.5, P2.6 to ~10,000 chars (consistent with Tier A) by removing the legacy formatting blocks and streamlining the schema specifications.
2. Instruct the LLM in the EXECUTION CONSTRAINTS block to limit JSON handoff objects to only the fields actually consumed by downstream steps. Verbose handoffs bloat context unnecessarily.
3. Consider whether SS-1/DD-1/WE-1 content is injected each time (via `{{PROMPT_DATA}}`). If so, this could be a major hidden context cost. These should be injected once at P1.1 and relied upon in context, not re-injected per step.

---

### Issue CX-8: Step Numbering Errors in P2.2, P2.3, P2.5, P2.6 (LOW — Clarity)

All four Tier B prompts jump from STEP 1 directly to STEP 3, with no STEP 2. STEP 2 existed in the old clarification-gate design and was removed during refactoring but the step numbers were not renumbered. This is confusing and could cause an LLM to infer a gap in its instructions.

**Recommendation:** Renumber STEP 3 → STEP 2 in P2.2, P2.3, P2.5, and P2.6.

---

## Part 2 — Per-Prompt Feedback

Sorted P1.1 → P2.13.

---

### P1.1 — Engagement Intake (5,527 chars)

**Objectives:** 3✓, 4✓, 5✓, 6✓, 7✓ | **Issues:** Obj 1 (minor), Obj 2 (N/A)

**Keep:**
- JSON-only output (no file generation). This is the right architectural choice — P1.1 establishes context, not a deliverable.
- Fail-fast validation with `P1_1_VALIDATION_ERRORS` output contract. Clean and correct.
- Canonical handoff pattern (`P1_1_CANONICAL_HANDOFF`) with structured 16-field context block.
- Explicit prohibition on inferring facts beyond what is provided. Prevents hallucinated intake data.
- DOWNSTREAM RULE clearly designating P1.1 as the single source of truth.

**Reduce:**
- The "DOWNSTREAM RULE" section and the Step 3 handoff instructions slightly overlap in conveying "downstream steps must use this JSON." One of the two is sufficient; the DOWNSTREAM RULE can be shortened to one sentence.

**Add:**
- A brief statement of the PM framework this workflow is aligned to (e.g. "This workflow follows PMI PMBOK 7th Edition planning process groups"). This anchors Objective 1 from the outset and sets expectations for all downstream outputs.
- Explicit note that SS-1/DD-1/WE-1 are injected here and will remain in context — downstream steps must not re-inject them.

**Remove:**
- Nothing needs removal.

---

### P1.2 — Stakeholder Analysis (15,774 chars)

**Objectives:** 1✓, 3✓, 4⚠, 5⚠, 6✓, 7✓ | **Issues:** Obj 4 (complex array formulas), Obj 5 (size)

**Keep:**
- Three-tier stakeholder inference framework (Extract → Infer → Override) with deduplication. This is excellent PM practice.
- Hard vs. soft minimums pattern: fail-fast on critical fields, open items on soft gaps. Correct approach.
- Inference depth floors by engagement size (8–15 / 15–25 / 25+). Prevents thin outputs.
- Insufficient-context guard with a specific user-facing error message. Good crash prevention.
- Transparency notice requirement for inferred stakeholders. Good governance practice.
- QA checklist before output. Strong quality control.
- `P1_2_CANONICAL_HANDOFF` for downstream chains.

**Reduce:**
- Sheet 2 (Stakeholder_Summary) Section 2 uses a TEXTJOIN+IF array formula to list names per quadrant. This is fragile in openpyxl and a common crash trigger. Replace with a simple COUNTIF per quadrant (counts only). Names can be found by filtering the register — they do not need to be concatenated programmatically.
- The tblAttentionItems (Section 3) logic is embedded in prose. Express it as a compact column specification to reduce ambiguity.

**Add:**
- Explicit guidance that `communication_cadence` for Manage_Closely stakeholders should align with the cadence defined in P2.9 (Communications Plan). This linkage is implied but not stated.

**Remove:**
- **STEP 2B heading ("USER CONFIRMATION GATE — MANDATORY; do NOT skip")** is actively misleading. The instruction body correctly says "Do NOT pause." The heading title contradicts this. Remove the heading; keep the instruction content under STEP 2 (or STEP 3 BUILD preamble).
- The `{{PROMPT_DATA}}` placeholder at the end if it is unused (see CX-2).

---

### P1.3 — Scope Statement (9,198 chars)

**Objectives:** 1✓, 2✓, 3✓, 4✓, 5✓, 6✓, 7✓ | **Issues:** Minor only

**Keep:**
- PMI scope management language (in-scope / out-of-scope / assumptions / constraints as distinct categories). Aligned to PMBOK Scope Management.
- SS-1 references instead of hardcoded formatting. Correct.
- JSON handoff output (`P1_3_CANONICAL_HANDOFF`).
- Step STEP 2 duplicate: there are two "STEP 2" occurrences — one appears to be a remnant. Resolve (see below).
- QA pattern consistent with P1.2.

**Reduce:**
- Step numbering: there are two occurrences of "STEP 2" in this prompt. The first appears to be a leftover from the clarification-gate phase. The second is the correct "USER CONFIRMATION" no-pause block. Remove the first or renumber so they are sequential.

**Add:**
- A statement linking scope assumptions and constraints to the Risk Register (P2.7). Scope assumptions with high impact-if-wrong are risk triggers — this cross-reference is implicit but should be explicit.

**Remove:**
- Nothing substantive. This is one of the cleanest prompts.

---

### P1.4 — Objectives & Success Criteria (13,159 chars)

**Objectives:** 1✓, 2✓, 3✓, 4✓, 5⚠, 6✓, 7✓ | **Issues:** Obj 5 (moderate size at 13k)

**Keep:**
- KPI/metric linkage to objectives. This is a core PM best practice (objectives must be measurable).
- SMART criteria application to success criteria. Keep this explicitly.
- SS-1 references and JSON handoff pattern.
- Reference to both P1.1 and P1.3 as inputs — correct dependency chain.

**Reduce:**
- At 13,159 chars this is reasonable but could tighten the Excel schema specification. The workbook columns are listed at high detail. Consider grouping the column spec as a compact table rather than a numbered prose list to save ~500–800 chars.

**Add:**
- Explicit cross-reference: success criteria defined here must become the acceptance criteria for deliverables listed in P2.1 (WBS). This linkage is critical for PM traceability and is currently only implied.
- Benefit realisation ownership: each objective should have an identified benefit owner (client-side). This is missing and is a gap against PMI benefit management practices.

**Remove:**
- Same STEP 2 duplicate issue as P1.3: two "STEP 2" occurrences. Resolve numbering.

---

### P1.5 — Engagement Charter (9,507 chars)

**Objectives:** 1✓, 2✓, 3✓, 4✓, 5✓, 6✓, 7✓ | **Issues:** Minor only

**Keep:**
- Sign-off gate pattern: the charter is correctly treated as a gate deliverable requiring formal sign-off before planning proceeds.
- References all four prior steps as inputs. Correct.
- JSON handoff output.
- Four distinct steps with clear no-pause instruction.

**Reduce:**
- At 9,507 chars this is well-sized. Minor tightening possible on the sign-off table specification — the instruction prose around it can be compressed.

**Add:**
- Explicit statement that the charter establishes the authority baseline for change control. Any change in scope, schedule, or budget after charter sign-off must go through P2.12. This authority linkage is absent.
- A "Supersedes" field in the charter header for version management (v1.0, v1.1 etc.).

**Remove:**
- Nothing substantive.

---

### P1.6 — Governance Framework (12,156 chars)

**Objectives:** 1✓, 2✓, 3✓, 4✓, 5⚠, 6✓, 7✓ | **Issues:** Obj 5 (moderate, 6 steps)

**Keep:**
- RACI integration with stakeholder register (P1.2). This linkage is essential.
- Governance body hierarchy (Steering → Project Management → Workstream) follows standard practice.
- Decision authority matrix. Critical PM artifact.
- SS-1 references and JSON handoff.

**Reduce:**
- Six steps is the highest step count in Phase 1. Some steps can be merged. Specifically, STEP 4 and STEP 5 (validation and QA) could be combined into a single QA block as done in P1.2.
- The RACI specification for governance bodies and the task-level RACI appear to be described separately in the prompt. Clarify which is in scope for P1.6 (governance RACI only) vs. P2.5 (task-level RACI). Any overlap creates duplicate work.

**Add:**
- An escalation path defined in the governance framework: what happens when a decision cannot be made at a given governance level. This is a standard governance gap.
- Explicit statement that governance meeting cadence defined here feeds directly into P2.2 (Schedule) and P2.9 (Communications Plan) as fixed calendar points.

**Remove:**
- Nothing substantive, but watch for RACI duplication with P2.5 (see Add point above).

---

### P2.1 — Work Breakdown Structure (10,497 chars)

**Objectives:** 1✓, 2✓, 3✓, 4✓, 5✓, 6✓, 7✓ | **Issues:** Minor only

**Keep:**
- PMBOK-aligned WBS decomposition hierarchy (Phase → Workstream → Deliverable → Work Package). Correct structure.
- WBS coding convention (1.1.1, 1.1.2 etc.). Keep the exact coding format specified.
- Deliverable register as a distinct sheet. This is a key downstream dependency for P2.2 and P2.11.
- Reference to all five prior steps as inputs. Correct.

**Reduce:**
- Four steps with some overlap between STEP 2 (validate) and STEP 4 (QA). These could merge.

**Add:**
- Work package effort estimates (days) should be captured here as a base for P2.2 (Schedule) and P2.6 (Budget). Currently these estimates are implied but not explicitly requested in the WBS prompt.
- An explicit statement that WBS IDs generated here must be used as the reference system in P2.2, P2.3, P2.4, P2.6, and P2.11. ID consistency is critical for traceability.

**Remove:**
- Nothing substantive.

---

### P2.2 — Schedule & Milestones (24,848 chars) ⚠ PRIORITY REWRITE

**Objectives:** 1✓, 2⚠, 3⚠, 4⚠, 5⚠, 6⚠, 7⚠ | **Issues:** Multiple high-priority issues

This is the most problematic prompt in the set. It is the largest (24,848 chars / ~6,200 tokens) and carries the most legacy issues.

**Keep:**
- Agile/Waterfall/Hybrid methodology adaptation logic (CONTEXT CHECK and methodology-specific sheet configuration). This is the most valuable unique content in this prompt and directly addresses Objective 1.
- Milestone Tracker specification (Sheet 1) — well structured.
- Sprint Plan sheet for Agile/Hybrid — correct PMI Agile Practice Guide alignment.
- Velocity Tracker table within Sprint Plan — good agile PM practice.
- The 5-sheet rationalisation instruction (cap at 5 sheets; no separate Gantt tab). Keep this.

**Reduce:**
- **Remove the full PwC formatting spec block** (~3,000 chars). Replace with: `Apply SS-1 (Word) and SS-1 (Excel) throughout.` This alone saves ~12% of the prompt length.
- The CONTEXT CHECK section (checking whether 8 of 16 fields are populated) is valuable but the specific field names listed are now defined in DD-1. The check can reference DD-1 field names rather than restating them inline.
- The WORD DOCUMENT STRUCTURE section (sections 1–11 plus appendices) is very detailed. Given that P2.2 is a scheduling document, the Word doc's primary consumer is the schedule Excel. The Word doc section could be reduced to the structure headings only, with SS-1 for formatting.

**Add:**
- `P2_2_CANONICAL_HANDOFF` JSON output (currently missing — see CX-3). Must include: milestone list with dates and IDs, schedule baseline date, delivery methodology, sprint details if applicable.
- Reference to P1_1_CANONICAL_HANDOFF and P2_1_CANONICAL_HANDOFF as the input sources. The current prompt still uses "Refer to documents generated in this thread" which is the legacy approach.

**Remove:**
- **STEP 2 heading gap**: The prompt jumps from STEP 1 to STEP 3. Renumber STEP 3 to STEP 2.
- **Internal contradiction**: The prompt says "you must first work through a structured scheduling clarification process with the user" in the preamble, then says "Do NOT pause or wait for user input" in STEP 1. Remove the clarification framing from the preamble; it is a legacy artefact.
- The mixed SS-1 token references and old hex codes. Where both appear, remove the hex codes and keep SS-1 tokens only.
- The Gantt bar generation via Excel conditional formatting. This is a known crash trigger in openpyxl and adds significant complexity for marginal visual value. Replace with a simpler Gantt approach: pre-populate date cells with the milestone or task duration as a value, and instruct the LLM to apply a single background colour fill via conditional formatting on column ranges only. Or remove the Gantt from the code-generated file and note it as a "user to configure" step.

---

### P2.3 — Critical Path Analysis (23,383 chars) ⚠ PRIORITY REWRITE

**Objectives:** 1✓, 2⚠, 3⚠, 4⚠, 5⚠, 6⚠, 7⚠ | **Issues:** Similar profile to P2.2

**Keep:**
- Forward and backward pass CPM calculation framework. This is technically correct PM methodology and is the core value of this prompt.
- Float calculation (total float / free float distinction). Keep.
- Dependency type support (FS, SS, FF, SF). Keep, as this is standard PMBOK schedule network analysis.
- Critical path identification logic and Zero Float as the identifier. Correct.

**Reduce:**
- Same as P2.2: remove the full legacy formatting spec block. Apply SS-1 references only.
- The CPM calculation instructions are detailed but some of the Excel formula specifications can be simplified. The forward/backward pass can be expressed as a calculation rule rather than a row-by-row instruction.

**Add:**
- `P2_3_CANONICAL_HANDOFF` JSON output (currently missing). Must include: critical path task IDs, total duration, zero-float tasks, and key schedule risk drivers. P2.7 (Risk Register) needs this.
- Explicit input reference to `P2_1_CANONICAL_HANDOFF` and `P2_2_CANONICAL_HANDOFF` (currently uses legacy "refer to documents" language).

**Remove:**
- STEP 2 gap: renumber STEP 3 to STEP 2.
- Clarification framing contradiction (same as P2.2).
- Old hex colour codes — use SS-1 tokens only.

---

### P2.4 — Resource Plan (10,188 chars)

**Objectives:** 1✓, 2✓, 3✓, 4✓, 5✓, 6✓, 7✓ | **Issues:** Minor only

**Keep:**
- Resource loading by role and workstream. Correctly structured for capacity planning.
- Over-allocation detection logic. Essential for realistic resource planning.
- Five steps with clear sequencing. Well structured.
- SS-1 references and JSON handoff.

**Reduce:**
- Minor: The step count (5) could be reduced to 3 by merging context-extraction steps.

**Add:**
- An explicit link between resource loading here and the budget model in P2.6. Resource-days × daily rates = labour cost. This linkage needs to be stated so the LLM populates resource rates in a format P2.6 can consume.
- A leave/availability factor (e.g. 0.8 utilisation assumption). Standard PM resourcing practice that is currently absent.

**Remove:**
- Nothing substantive.

---

### P2.5 — RACI Matrix (21,769 chars) ⚠ PRIORITY REWRITE

**Objectives:** 1✓, 2⚠, 3⚠, 4⚠, 5⚠, 6⚠, 7⚠ | **Issues:** Similar profile to P2.2/P2.3

**Keep:**
- Exactly-one-A-per-row rule for the RACI. This is the correct PM governance constraint and prevents ambiguous accountability.
- Role-based RACI rows (not person-based). Correct for an engagement-level RACI.
- Cross-reference to governance RACI from P1.6. Important consistency check.
- Three-deliverable group structure (WBS deliverables, governance activities, management activities). Comprehensive coverage.

**Reduce:**
- Remove the legacy formatting spec block. Apply SS-1 only.
- The step count (STEP 1 → STEP 3) is missing STEP 2; renumber.
- Reduce the column specification to a compact table rather than prose. This alone saves ~1,000–2,000 chars.

**Add:**
- `P2_5_CANONICAL_HANDOFF` JSON output (currently missing). Must include the full RACI as a structured array for P2.9 (Communications) and P2.13 (PMO Playbook) to consume without re-parsing a Word/Excel file.
- Explicit input reference to `P1_2_CANONICAL_HANDOFF` (stakeholders), `P1_5_CANONICAL_HANDOFF` (charter roles), `P1_6_CANONICAL_HANDOFF` (governance RACI), and `P2_1_CANONICAL_HANDOFF` (WBS deliverables).

**Remove:**
- STEP 2 gap: renumber STEP 3 to STEP 2.
- Clarification framing contradiction.
- Old hex colour codes.

---

### P2.6 — Budget & Cost Baseline (26,477 chars) ⚠ PRIORITY REWRITE

**Objectives:** 1✓, 2⚠, 3⚠, 4⚠, 5⚠, 6⚠, 7⚠ | **Issues:** Largest prompt; multiple high-priority issues

This is the longest prompt in the set (26,477 chars / ~6,619 tokens). It is also the most financially sensitive.

**Keep:**
- Earned Value Management (EVM) integration (Planned Value, Earned Value, Cost Performance Index, Schedule Performance Index). This is advanced but correct PMBOK financial tracking methodology.
- Contingency reserve distinction from management reserve. Important PM cost management concept.
- Cost baseline vs. cost budget distinction. Correct.
- Fee structure type adaptation (fixed fee, T&M, capped T&M, milestone-based). Good commercial coverage.

**Reduce:**
- Remove the full legacy formatting spec. Apply SS-1 only. This saves ~3,000 chars.
- The EVM section is comprehensive but the formula specifications (CPI = EV/AC etc.) are well known — they can be stated as a formula list rather than described in prose.
- Reduce the Excel sheet count. Six or more sheets in a single file increases crash probability significantly. Aim for 5 sheets maximum (consistent with P2.2's rationalisation instruction).

**Add:**
- `P2_6_CANONICAL_HANDOFF` JSON output (currently missing). Must include: total budget, contingency reserve, management reserve, fee structure type, and cost breakdown by workstream. P2.13 (PMO Playbook) needs this.
- Explicit input reference to `P2_1_CANONICAL_HANDOFF` (WBS work packages as cost holders) and `P2_4_CANONICAL_HANDOFF` (resource rates and effort days).
- A currency declaration at the top of the prompt (draw from `P1_1_CANONICAL_HANDOFF`). Currency inconsistency in multi-currency engagements is a common error.

**Remove:**
- STEP 2 gap: renumber STEP 3 to STEP 2.
- Clarification framing contradiction.
- Old hex colour codes.

---

### P2.7 — Risk Register (11,025 chars)

**Objectives:** 1✓, 2✓, 3✓, 4✓, 5✓, 6✓, 7✓ | **Issues:** Minor only

**Keep:**
- 5×5 risk matrix (Likelihood × Impact). Standard PMI/ISO 31000 aligned.
- Risk owner assignment per risk. Correct accountability model.
- Mitigation vs. contingency distinction. Correct PM risk response categories.
- Residual risk after mitigation. Good PM practice.
- JSON handoff and no-pause instruction.

**Reduce:**
- Seven steps is the most in any single prompt. Steps 5 and 6 (QA and output contract) can be merged into a single step as in other prompts.

**Add:**
- Explicit linkage: risks with Critical impact affecting milestones on the critical path (from P2.3) should be flagged as schedule risks. This cross-reference needs to be stated.
- A risk appetite statement drawn from the engagement charter (P1.5). Without an explicit risk appetite, threshold ratings (High / Medium / Low) are arbitrary.

**Remove:**
- Step numbering irregularity: STEP 2 appears twice in the step sequence. Deduplicate.

---

### P2.8 — Issue & Dependency Log (13,210 chars)

**Objectives:** 1✓, 2✓, 3✓, 4✓, 5⚠, 6✓, 7✓ | **Issues:** Obj 5 (moderate size at 13k; six steps)

**Keep:**
- Issue vs. dependency as separate registers. Correct PM practice (dependencies are forward-looking constraints; issues are current blockers).
- Escalation path in issue log. Important.
- Dependency type taxonomy (finish-to-start, start-to-start, etc.). Consistent with P2.3 CPM.
- JSON handoff and no-pause instruction.

**Reduce:**
- Six steps. Merge steps 5 (QA) and 6 (output) into one as in other prompts.
- The dependency log specification mirrors P2.3 (Critical Path) significantly. Reduce to a reference: "Dependency types and predecessor/successor logic must be consistent with the dependency taxonomy in P2_3_CANONICAL_HANDOFF."

**Add:**
- A cross-reference to the Risk Register (P2.7): any unresolved issue with High impact should trigger a risk entry in the risk register. This escalation linkage is absent.

**Remove:**
- Nothing substantive.

---

### P2.9 — Communications Plan (13,375 chars)

**Objectives:** 1✓, 2✓, 3✓, 4✓, 5⚠, 6✓, 7✓ | **Issues:** Obj 5 (moderate at 13k), no-pause missing

**Keep:**
- Stakeholder-linked communications matrix. Correct — comms plan must be driven by stakeholder analysis.
- Communication frequency alignment with governance cadence (from P1.6). Important linkage.
- Restricted access notation for sensitive stakeholder comms. Good governance.
- JSON handoff.

**Reduce:**
- Six steps. Merge QA and output into one.

**Add:**
- No-pause instruction (currently absent — see CX-4).
- Explicit link to governance meeting cadence from `P1_6_CANONICAL_HANDOFF` as fixed anchor points in the communications calendar.
- A communication channel specification tied to each stakeholder's preferred channel from `P1_2_CANONICAL_HANDOFF`. The prompt currently infers channels from scratch rather than reading the confirmed preferences from P1.2.

**Remove:**
- Nothing substantive.

---

### P2.10 — Status Reporting Template (11,781 chars)

**Objectives:** 1✓, 2✓, 3✓, 4⚠, 5⚠, 6✓, 7⚠ | **Issues:** Obj 4/7 (three file types in one code call)

**Keep:**
- Red / Amber / Green status framework. Standard PM reporting convention.
- Escalation threshold definition in status report. Good.
- Word + Excel combination (narrative + tracker). Appropriate for a status reporting template.
- JSON handoff.

**Reduce:**
- Remove the PowerPoint requirement from this prompt. P2.10 asks for Word + Excel + PowerPoint — three files in a single code interpreter call. PowerPoint generation via `python-pptx` alongside `python-docx` and `openpyxl` in a single call is the highest crash-risk scenario in the entire prompt set. The status report PowerPoint (a steering committee deck template) is more appropriately generated as a section in P2.13 (PMO Playbook) which already produces a PowerPoint.

**Add:**
- No-pause instruction (currently absent — see CX-4).
- An explicit statement that the status report template uses the milestone IDs and workstream structure from `P2_2_CANONICAL_HANDOFF`, so the template is pre-populated with the correct milestone names, not generic placeholders.

**Remove:**
- The PowerPoint output from this prompt (see Reduce above). Redirect it to P2.13.

---

### P2.11 — Quality Plan (10,722 chars)

**Objectives:** 1✓, 2✓, 3✓, 4✓, 5✓, 6✓, 7✓ | **Issues:** Minor only; no-pause missing

**Keep:**
- Quality gate framework tied to WBS deliverables. PMI-aligned.
- Deliverable review schedule structure. Important for formal QA.
- Non-conformance log. Good practice.
- JSON handoff.

**Reduce:**
- Four steps with some potential merging at steps 3 and 4.

**Add:**
- No-pause instruction (currently absent — see CX-4).
- A cross-reference: quality gates defined here should appear as milestones in P2.2 (Schedule). This linkage is currently missing and could cause quality gates to be invisible in the project schedule.
- Quality acceptance criteria should explicitly reference the success criteria from `P1_4_CANONICAL_HANDOFF` as the authoritative standard for "done."

**Remove:**
- Nothing substantive.

---

### P2.12 — Change Control Plan (11,656 chars)

**Objectives:** 1✓, 2✓, 3✓, 4✓, 5✓, 6✓, 7✓ | **Issues:** Minor only; no-pause missing

**Keep:**
- Change request → assessment → approval → implementation flow. PMI-aligned change control process.
- Change impact assessment across scope, schedule, budget, and quality. Correct.
- Change log in the workbook. Essential audit trail.
- Reference to charter as the baseline against which changes are assessed.
- JSON handoff.

**Reduce:**
- Five steps is appropriate. No reduction needed.

**Add:**
- No-pause instruction (currently absent — see CX-4).
- An explicit instruction that emergency change approval bypasses the normal committee cycle but still requires documentation. Emergency change paths are a standard gap in change control plans.
- Cross-reference: approved scope changes here must trigger updates to P2.2 (Schedule), P2.6 (Budget), and P2.1 (WBS). This impacts three downstream registers and this chain should be stated.

**Remove:**
- Nothing substantive.

---

### P2.13 — PMO Playbook (11,071 chars)

**Objectives:** 1✓, 2✓, 3✓, 4⚠, 5✓, 6✓, 7⚠ | **Issues:** Obj 4/7 (PowerPoint + Excel in one call)

This prompt has been dramatically improved from its original 41,334-char version. The reduction to 11,071 chars is a significant improvement.

**Keep:**
- Consolidation review across all 18 prior inputs before building. This is the right sequencing for a capstone deliverable.
- Readiness assessment framework (Ready / Proceed with Conditions / Not Ready). Strong.
- Consistency check across scope, schedule, resource, budget, RACI, and risk. Good.
- JSON handoff.

**Reduce:**
- Four steps is appropriate. The consolidation review (STEP 1) could be tightened if it relies on canonical JSON handoff objects rather than re-parsing "documents in thread."

**Add:**
- No-pause instruction (currently absent — see CX-4).
- Explicit instruction that all data for the playbook must be read from canonical handoff JSON objects (P1_1 through P2_12). If any handoff JSON is missing, the LLM should list which steps need to be re-run rather than inferring content.
- A delivery team distribution list as part of the playbook output (drawn from P1.2 stakeholder register). Currently the playbook consolidates planning artifacts but doesn't define its own distribution.

**Remove:**
- Consolidate the PowerPoint and Excel into a single code call explicitly. The prompt implies both outputs but doesn't state they must be in one script. Add: "Generate both output files within a single code interpreter call using one Python script."
- Remove any residual reference to "refer to documents generated in this message thread." All inputs must come from canonical JSON handoffs.

---

## Part 3 — Priority Matrix

| Priority | Prompt(s) | Issue | Objective(s) |
|---|---|---|---|
| **Critical** | P2.2, P2.3, P2.5, P2.6 | Complete Tier B → Tier A refactoring (remove old format spec, add JSON handoff, fix step numbering, remove contradiction) | 2, 3, 5, 6, 7 |
| **Critical** | P2.2, P2.3, P2.5, P2.6 | Trim to ~10,000 chars each | 4, 5 |
| **Critical** | All | Add rule 10 to EXECUTION CONSTRAINTS: suppress code/reasoning display | 6 |
| **High** | P2.10 | Remove PowerPoint from this prompt; move to P2.13 | 4, 7 |
| **High** | P1.2 | Remove misleading STEP 2B heading; simplify array formulas | 4, 6 |
| **High** | All (except P1.1) | Clarify `{{PROMPT_DATA}}` behaviour or remove | 3, 5 |
| **High** | P2.10, P2.11, P2.12, P2.13 | Add no-pause instruction | 4, 5 |
| **Medium** | P2.13 | Enforce single-call for PowerPoint + Excel explicitly | 7 |
| **Medium** | P1.1 | Add PM framework citation (e.g. PMBOK 7) | 1 |
| **Medium** | P2.7 | Fix duplicate STEP 2; add risk appetite linkage to charter | 1, 4 |
| **Medium** | P1.4 | Add benefit realisation ownership; fix STEP 2 duplicate | 1 |
| **Medium** | P1.6 | Add escalation path; clarify RACI scope vs P2.5 | 1, 3 |
| **Medium** | P2.4 | Add utilisation factor; link resource rates to P2.6 | 1, 3 |
| **Low** | P1.3, P1.4 | STEP 2 duplicate — renumber | 4 (clarity) |
| **Low** | P2.7, P2.8 | Merge QA + output steps to reduce step count | 5 |
| **Low** | P1.5 | Add supersedes/version field to charter | 2 |
| **Low** | P2.11 | Link quality gates to P2.2 schedule milestones | 1, 3 |
| **Low** | P2.12 | Add emergency change approval path | 1 |

---

## Summary Scorecard

| Prompt | Obj 1 | Obj 2 | Obj 3 | Obj 4 | Obj 5 | Obj 6 | Obj 7 |
|---|---|---|---|---|---|---|---|
| P1.1 | ⚠ | — | ✓ | ✓ | ✓ | ✓ | ✓ |
| P1.2 | ✓ | ✓ | ✓ | ⚠ | ⚠ | ✓ | ✓ |
| P1.3 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| P1.4 | ⚠ | ✓ | ✓ | ✓ | ⚠ | ✓ | ✓ |
| P1.5 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| P1.6 | ⚠ | ✓ | ✓ | ✓ | ⚠ | ✓ | ✓ |
| P2.1 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| P2.2 | ✓ | ✗ | ✗ | ⚠ | ✗ | ✗ | ✗ |
| P2.3 | ✓ | ✗ | ✗ | ⚠ | ✗ | ✗ | ✗ |
| P2.4 | ⚠ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| P2.5 | ✓ | ✗ | ✗ | ⚠ | ✗ | ✗ | ✗ |
| P2.6 | ✓ | ✗ | ✗ | ⚠ | ✗ | ✗ | ✗ |
| P2.7 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| P2.8 | ✓ | ✓ | ✓ | ✓ | ⚠ | ✓ | ✓ |
| P2.9 | ✓ | ✓ | ✓ | ✓ | ⚠ | ✓ | ✓ |
| P2.10 | ✓ | ✓ | ✓ | ✗ | ⚠ | ✓ | ✗ |
| P2.11 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| P2.12 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| P2.13 | ✓ | ✓ | ✓ | ⚠ | ✓ | ✓ | ⚠ |

**Key:** ✓ = passes | ⚠ = passes with minor gaps | ✗ = fails

**Immediate action required:** P2.2, P2.3, P2.5, P2.6 (multiple ✗), P2.10 (Obj 4 and 7 ✗)
