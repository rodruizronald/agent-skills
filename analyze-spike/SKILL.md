---
name: analyze-spike
description: Conducts deep-dive investigation for spike tickets, producing structured findings and actionable recommendations rather than implementation plans
---

Investigate spike tickets with the rigor of a staff engineer conducting a technical deep-dive. Extract research questions, systematically investigate each one using the codebase and available context, evaluate options with evidence, and produce a findings document that gives the team confidence to make decisions.

## How to Use This Skill

The user will provide:

- A spike ticket (pasted text, screenshot, or URL)
- Optionally, relevant code files, architecture docs, codebase context, or external references
- Optionally, constraints or hypotheses the team already has

Investigate the spike through four sequential phases and produce a structured findings document.

## Mission

Ensure no spike is closed without:

- A clear articulation of what questions the spike was trying to answer
- Evidence-based findings for each question, grounded in code and documentation
- An honest assessment of what was learned and what remains uncertain
- Actionable recommendations that enable the team to make informed decisions
- Clearly defined follow-up work (tickets, further spikes, or decisions needed)

## Core Principles

1. **Curiosity over confirmation** — Investigate to discover truth, not to validate assumptions. If evidence contradicts the team's hypothesis, say so.
2. **Evidence over opinion** — Every finding must reference specific code, documentation, benchmarks, or external sources. "I believe" is not a finding.
3. **Explicit uncertainty** — Distinguish clearly between what was proven, what is likely, and what remains unknown. Partial answers are valuable when labeled honestly.
4. **Depth over breadth** — A spike that thoroughly answers 3 questions is more valuable than one that superficially touches 10.
5. **Actionable outputs** — Every finding must connect to a decision the team can make or an action they can take.
6. **Know when to stop** — A spike is not open-ended research. Define the investigation boundary and respect it.

## Workflow

Execute these phases sequentially. Do not skip phases. Each phase builds on the previous.

---

### PHASE 1: Spike Comprehension

#### 1.1 Extract Research Questions

Parse the spike ticket to identify:

- **Primary question**: The single core question the spike exists to answer.
- **Secondary questions**: Supporting questions that inform the primary answer.
- **Success criteria**: How the team will know the spike is "done" — what decisions should be unblocked by the findings.
- **Investigation boundary**: What is explicitly out of scope for this spike.

#### 1.2 Classify Each Question

For each research question, determine:

- **Question type**: Feasibility ("Can we do X?") | Comparison ("Should we use A or B?") | Discovery ("How does X work?") | Risk Assessment ("What could go wrong with X?") | Estimation ("How much effort/cost/time for X?")
- **Evidence needed**: What kind of evidence would answer this — code analysis, benchmarks, PoC, external research, expert consultation?
- **Confidence target**: What level of confidence does the team need — directional (rough sense) | informed (solid understanding) | proven (demonstrated with evidence/PoC)?

#### 1.3 Identify Ticket Gaps

Flag: vague investigation goals ("look into", "explore", "research"), missing success criteria, unbounded scope, conflicting constraints, assumptions baked into the questions that should be validated first.

**Phase 1 Output Format:**

```
## Spike Comprehension Summary

### Primary Question
[Single sentence — the core question this spike must answer]

### Research Questions
| ID | Question | Type | Evidence Needed | Confidence Target |
|----|----------|------|-----------------|-------------------|
| RQ1 | [question] | Feasibility/Comparison/Discovery/Risk/Estimation | [evidence type] | Directional/Informed/Proven |

### Success Criteria
- [ ] [What decision this spike unblocks]
- [ ] [What the team should be able to do after reading the findings]

### Investigation Boundary
- In scope: [what will be investigated]
- Out of scope: [what will NOT be investigated]

### Ticket Quality Issues
- [ ] [Issue]: [Description and why it matters]
```

---

### PHASE 2: Investigation

This is the core of the spike. For each research question, conduct a systematic investigation.

#### 2.1 Codebase Investigation

Where relevant, examine:

- **Current implementation**: How does the relevant system work today? Trace the actual code paths.
- **Patterns and conventions**: What architectural patterns are established? What would a solution need to conform to?
- **Technical constraints**: Framework limitations, library versions, API contracts, performance characteristics.
- **Hidden complexity**: Undocumented behavior, implicit dependencies, edge cases buried in the code.
- **Prior art**: Has something similar been attempted before? Are there TODOs, commented-out code, or abandoned approaches that reveal past decisions?

#### 2.2 External Research

Where relevant, investigate:

- **Documentation**: Official docs for relevant libraries, frameworks, APIs, or services.
- **Known solutions**: How have others solved this problem? What patterns exist?
- **Limitations and gotchas**: Known issues, version compatibility concerns, deprecation timelines.
- **Benchmarks and data**: Performance characteristics, cost data, scaling limits — from credible sources.

#### 2.3 Evidence Gathering

For each research question, collect:

- **Direct evidence**: Code snippets, configuration, documentation excerpts, benchmark results that directly answer the question.
- **Contextual evidence**: Related findings that inform the answer without directly answering it.
- **Counter-evidence**: Anything that complicates or contradicts the emerging answer — do not suppress inconvenient findings.

**Phase 2 Output Format (per research question):**

```
## Investigation: [RQ ID] — [Question]

### Approach
[How this question was investigated — what was examined, searched, or tested]

### Findings

#### Finding 1: [Title]
- **Evidence**: [Specific code reference, doc link, benchmark data, etc.]
- **Implication**: [What this means for the question]

#### Finding 2: [Title]
- **Evidence**: [...]
- **Implication**: [...]

[Repeat as needed]

### Counter-Evidence / Complications
- [Anything that complicates the answer]

### Confidence Assessment
- Confidence level: High / Medium / Low
- What would increase confidence: [specific action — e.g., "run a PoC", "benchmark under production load", "consult with platform team"]
```

---

### PHASE 3: Analysis & Synthesis

#### 3.1 Answer Each Research Question

For each question, provide:

- **Direct answer**: A clear, concise answer to the question as asked.
- **Supporting evidence summary**: The key evidence that supports this answer.
- **Caveats and conditions**: Under what conditions does this answer hold? What assumptions is it built on?
- **Confidence level**: High (proven with evidence) | Medium (strong indicators, some gaps) | Low (directional only, significant uncertainty remains).

#### 3.2 Options Analysis (if applicable)

When the spike involves choosing between approaches:

| Criteria        | Option A: [Name] | Option B: [Name] | Option C: [Name] |
| --------------- | ---------------- | ---------------- | ---------------- |
| [Criterion 1]   | [Assessment]     | [Assessment]     | [Assessment]     |
| [Criterion 2]   | [Assessment]     | [Assessment]     | [Assessment]     |
| Effort estimate | [Estimate]       | [Estimate]       | [Estimate]       |
| Risk level      | [Level]          | [Level]          | [Level]          |
| Reversibility   | [Easy/Hard]      | [Easy/Hard]      | [Easy/Hard]      |

For each option: strengths, weaknesses, risks, and unknowns.

#### 3.3 Risk & Unknowns Register

| Item   | Type           | Severity     | Status          | Mitigation / Next Step |
| ------ | -------------- | ------------ | --------------- | ---------------------- |
| [item] | Risk / Unknown | High/Med/Low | Resolved / Open | [action]               |

#### 3.4 Recommendation

- **Recommended approach**: [Clear statement]
- **Rationale**: [Why this option, grounded in findings]
- **Key tradeoffs accepted**: [What you're giving up and why it's acceptable]
- **Conditions / prerequisites**: [What must be true for this recommendation to hold]

If the evidence does not support a clear recommendation, say so explicitly. "The investigation was inconclusive because [reason]. Further work is needed: [specific next steps]." An honest "we don't know yet" is a valid spike outcome.

**Phase 3 Output Format:**

```
## Analysis & Synthesis

### Research Question Answers

#### RQ1: [Question]
- **Answer**: [Direct, concise answer]
- **Key evidence**: [Summary of supporting evidence]
- **Caveats**: [Conditions and assumptions]
- **Confidence**: High / Medium / Low

[Repeat for each RQ]

### Options Analysis (if applicable)
[Options comparison table and narrative]

### Risk & Unknowns Register
[Table of risks and open unknowns]

### Recommendation
- **Recommended approach**: [statement]
- **Rationale**: [evidence-based justification]
- **Tradeoffs accepted**: [what you're giving up]
- **Prerequisites**: [what must be true]
```

---

### PHASE 4: Outputs & Follow-Up

#### 4.0 Output Location

The findings document MUST be saved as a markdown file at `./spike/SYS_[NUMBER]_FINDINGS.md`, where `[NUMBER]` is the ticket number extracted from the ticket ID or branch name (e.g., `./spike/SYS_820_FINDINGS.md`). If the `./spike/` directory does not exist, create it first. If the ticket number cannot be determined, ask the user for it before saving.

#### 4.1 Executive Summary

Write a concise summary (5–8 sentences max) that a PM or Tech Lead can read in 30 seconds and understand: what was investigated, what was found, what is recommended, and what happens next.

#### 4.2 Follow-Up Work

Define concrete follow-up items:

- **Implementation tickets**: If the spike recommends building something, outline the tickets that should be created. These are NOT detailed implementation plans — they are ticket descriptions with enough context to be planned (potentially using the `plan-ticket` skill).
- **Further spikes**: If questions remain unanswered, define focused follow-up spikes with clear scoping.
- **Decisions needed**: If the recommendation requires stakeholder approval, define exactly what decision is needed, who makes it, and what information they need.
- **Timeboxed PoCs**: If a finding needs validation, define the PoC scope, success criteria, and timebox.

#### 4.3 Appendix

Include raw evidence that supports the findings but would clutter the main document: code snippets examined, benchmark data, configuration details, links to external resources, alternative approaches considered and discarded (with reasons).

**Phase 4 Output Format:**

```
## Executive Summary
[5–8 sentence summary for stakeholders]

## Follow-Up Work

### Implementation Tickets to Create
| Ticket Title | Description | Priority | Depends On |
|--------------|-------------|----------|------------|
| [title] | [brief description with context] | P0/P1/P2 | [dependency] |

### Further Spikes Needed
| Spike Title | Question to Answer | Timebox | Why Now |
|-------------|-------------------|---------|---------|
| [title] | [focused question] | [hours/days] | [what it unblocks] |

### Decisions Needed
| Decision | Decision Maker | Options | Deadline |
|----------|---------------|---------|----------|
| [what] | [who] | [A / B / C] | [when] |

### Appendix
[Raw evidence, code snippets, links, discarded alternatives]
```

---

## Output Structure

### Standard Output (investigation complete):

Produce the full findings document: Executive Summary → Phase 1 → Phase 2 → Phase 3 → Phase 4. Save as a markdown file in `./spike/`.

End with two suggested next steps:

- **"Plan Implementation"**: "The spike recommends [approach]. I can help plan the implementation tickets identified in the follow-up section using the `plan-ticket` skill."
- **"Investigate Further"**: "There are [N] open unknowns. I can help scope a focused follow-up spike to address [specific unknown]."

### Partial Output (investigation blocked or inconclusive):

If the investigation cannot be completed due to missing context, access limitations, or genuinely inconclusive evidence:

```
⚠️ INVESTIGATION INCOMPLETE — [REASON]

This spike could not be fully resolved.

### What Was Answered
- [RQ that were answered, with brief findings]

### What Remains Unanswered
- [RQ that could not be answered]
- Reason: [why — missing code context, need PoC, need expert input, etc.]
- What would resolve it: [specific action]

### Partial Recommendation
[If enough was learned to give directional guidance, provide it with appropriate caveats]
```

---

## Behavioral Guidelines

1. **Investigate, don't plan** — This skill produces findings, not implementation plans. If the user needs a plan after the spike, direct them to `plan-ticket`.
2. **Follow the evidence** — If findings contradict the team's initial hypothesis or the ticket's assumptions, report it clearly. The spike exists to find truth, not to confirm expectations.
3. **Show your work** — Reference specific files, code paths, documentation, and data. "I looked at the code and it seems fine" is not a finding.
4. **Quantify where possible** — "It's slow" vs "The query takes ~800ms under load based on [evidence]". Prefer numbers over adjectives.
5. **Distinguish fact from inference** — "The code does X" (fact) vs "This likely means Y" (inference). Both are valuable; conflating them is dangerous.
6. **Respect the timebox** — Spikes have boundaries. If a question is pulling toward a rabbit hole, note it as a follow-up rather than diving in.
7. **Write for the absent reader** — The findings document will be read by people who weren't part of the investigation. Include enough context that someone can understand the findings without having been there.
8. **Connect findings to decisions** — Every finding should map to either a decision the team can now make or a follow-up action. Orphan findings that don't connect to anything actionable should be moved to the appendix or cut.
9. **Be honest about confidence** — A low-confidence finding that is labeled as such is far more valuable than a low-confidence finding presented as certain.

## Error Handling

**Insufficient ticket information**: Complete Phase 1 only. End with a "Clarification Needed" section listing what information is missing to begin investigation, who needs to provide it, and what questions it would enable answering.

**No codebase context provided**: Ask the user to share relevant files, directory structure, or architecture documentation. If the spike is purely about external research (e.g., evaluating a third-party tool), proceed without codebase context and note the limitation. If the spike requires code investigation, flag the missing context and explain what specifically is needed.

**Contradictory evidence found**: Document both sides explicitly. Do NOT pick a side without strong justification. Present the contradiction as a finding and recommend how to resolve it (further investigation, PoC, or escalation to a decision maker).

**Investigation hits a dead end**: Document what was tried, why it didn't yield results, and what alternative approaches might work. A documented dead end is a valid finding — it prevents others from repeating the same investigation.

**Scope creep during investigation**: If the investigation reveals that the spike's scope is insufficient to answer the primary question, stop and flag it. Recommend rescoping the spike rather than silently expanding the investigation boundary.
