# tulla-agent — Ontology-Driven Coding-Agent Orchestrator

**tulla-agent** is an ontology-driven coding-agent orchestrator: a fleet of
phase agents takes software from idea to committed code, coordinating through
a shared RDF knowledge graph — architecture decisions as first-class
resources, SHACL-validated phase handoffs, drift-resistant structured
context, and end-to-end traceability from commit to quality attribute.
Multiple agent instances, one semantic brain.

> This fleet is the **canonical Tulla artifact**. The Python engine in
> [`tulla`](../tulla) is the reference implementation; the shared semantic
> substrate (ontology server, pipeline ontologies, gate shapes) lives in
> [`semantic-tool-use`](../semantic-tool-use).

The pipeline runs up to 18 sub-phase agents: **D1–D5 (discovery) → R1–R6
(research, optional) → P1–P6 (planning) → I1 (implementation loop)**.

Each sub-phase agent reads upstream facts from the RDF knowledge graph, does
its analysis, writes SHACL-validated phase facts back to the graph (rollback
on violation — see gates below), and signals completion. The orchestrator
sequences agents by querying the phases graph after each completion and
mechanically blocks on failed coverage gates.

## Architecture

```
tulla (orchestrator)
│
├── Discovery
│   ├── D1_northstar          claude-opus-4-7   Context & Ecosystem Inventory
│   ├── D2_persona_exploration claude-sonnet-5 Persona Discovery (JTBD, pain points)
│   ├── D3_scoring            claude-sonnet-5  Value Mapping (effort vs impact)
│   ├── D4_gap_analysis       claude-opus-4-7   Gap Analysis (iSAQB quality attributes)
│   └── D5_research_brief     claude-sonnet-5  Integration / Research Brief [gate]
│
├── Research (optional — only if D5 mode=research)
│   ├── R1_question_refinement claude-sonnet-5 Question Refinement (novelty check)
│   ├── R2_source_gathering   claude-sonnet-5  Source Identification
│   ├── R3_investigation      claude-sonnet-5  Source Investigation
│   ├── R4_synthesis          claude-opus-4-7   Literature Review & Synthesis
│   ├── R5_experiments        claude-sonnet-5  Prototype Experiments (Bash enabled)
│   └── R6_recommendation     claude-sonnet-5  Research Recommendation [gate]
│
├── Planning
│   ├── P1_requirements       claude-sonnet-5  Discovery Context Integration
│   ├── P2_architecture       claude-sonnet-5  Codebase Analysis
│   ├── P3_design_decisions   claude-opus-4-7   Architecture & ADRs [gate]
│   ├── P4_implementation_plan claude-opus-4-7   Implementation Plan
│   └── P5_risk_assessment    claude-sonnet-5  Research Requests & Risk Assessment
│
└── Implementation
    └── I1_coding             claude-opus-4-7   Orchestrates claude_code sub-agent [gate]
        └── claude_code       claude-native     Actual coding, tests, commit
```

### Model assignments

| Agent(s) | Model | Rationale |
|----------|-------|-----------|
| D1, D4, R4, P3, P4, I1 | `claude-opus-4-7` | Deep analysis, architecture, complex synthesis |
| D2, D3, D5, R1, R2, R3, R5, R6, P1, P2, P5 | `claude-sonnet-5` | Structured extraction, source work, planning |
| claude_code | `claude-native` | Direct file editing with auto permission mode |

### SHACL gates

**Every phase is gated.** Each agent persists via `record_phase_result_tool`,
which SHACL-validates the result server-side and rolls back all triples on
non-conformance. The agent contract is uniform: do NOT report completion until
`ok: true`; fix per the reported violations and retry (max 3 attempts); after
that, FAIL loudly quoting every violation verbatim. No agent may downgrade a
validation failure to a warning.

| Agent | Gate shape | Key required fields |
|-------|-----------|----------------|
| D1_northstar | D1OutputShape * | key_capabilities, idea_verbatim_features, ecosystem_context |
| D2_persona_exploration | D2OutputShape * | personas, primary_persona, non_negotiable_needs |
| D3_scoring | D3OutputShape * | total_value_score, quadrant, complexity_tier, verdict |
| D4_gap_analysis | D4OutputShape * | gaps, quality_attribute_gaps, root_blocker |
| D5_research_brief | D5OutputShape | mode, recommendation, northstar, mandatory_features, key_constraints |
| R1_question_refinement | R1OutputShape | questions_refined, research_questions |
| R6_recommendation | R6OutputShape | findings_count, recommendation, synthesised_answers, risks |
| P1_requirements | P1OutputShape * | feature_scope, mandatory_feature_coverage, uncovered_mandatory_features |
| P2_architecture | P2OutputShape * | codebase_summary, relevant_modules |
| P3_design_decisions | P3OutputShape | total_dependencies, circular_dependencies, feature_coverage, uncovered_features |
| P4_implementation_plan | P4OutputShape * | tasks (with per-task features), feature_coverage, uncovered_features |
| P5_risk_assessment | P5OutputShape * | research_requests, readiness_verdict |
| P6_prd_export | P6OutputShape * | requirement_count, prd_context, coverage_gate, uncovered_features |
| I1_coding | LWTraceOutputShape | change_type, affected_files, conformance_assertion, commit_ref, change_summary, timestamp |

All shapes are defined in `semantic-tool-use`'s
`tulla/ontologies/phase-content.trig` and auto-seeded into the phases graph at
server startup. Shapes marked `*` were added together with this fleet contract.
Server-side enforcement details: the P1/P3/P4 shapes carry
`sh:hasValue "[]"` on their `uncovered_*` fields, so a phase that drops a
feature is **rolled back**, not just flagged; a phase whose declared gate shape
cannot be resolved (or whose server has no validator) FAILS instead of passing
silently. P6's `coverage_gate` is deliberately recordable as `"fail"` so the
orchestrator and I1 can see and block on it.

### Feature-coverage gate (mechanical drift check)

Prompt discipline alone does not prevent feature drop. The pipeline carries a
machine-checkable coverage chain from discovery to code:

```
D5 mandatory_features            (the contract, idea's own words)
  → P1 feature_scope             + mandatory_feature_coverage / uncovered_mandatory_features
    → P3 feature_coverage        (feature_id → ADRs/components)
      → P4 tasks[].features      + feature_coverage / uncovered_features
        → P6 prd:coversFeature   per requirement + Step 2b recall_facts verification
          → I1 pre-flight        aborts unless coverage_gate="pass"
```

Enforcement points:
- **P1** blocks itself while `uncovered_mandatory_features ≠ []`.
- **P6** re-reads what was actually stored (`recall_facts(predicate="prd:coversFeature")`)
  and compares against P1 feature_scope — a fail is recorded to the graph, not hidden.
- **The orchestrator** routing table blocks on `p1.uncovered_mandatory_features ≠ []`
  and on `p6.coverage_gate = "fail"` — a literal value comparison, no judgement.
- **I1** independently re-checks the P6 gate at pre-flight before dispatching any coder.

### Human-in-the-loop gate points (HITL)

Phases whose definitions carry `phase:requiresApproval "true"` (ship defaults:
**d5, p1, p6** — toggleable at `/dashboard/settings/phases`) record their
validated output with `phase:approvalStatus "pending"`. A pending output does
**not** count as complete: the orchestrator's Query A filters it out, and a
Query D pre-check runs before the routing table:

- **pending** → the orchestrator long-polls `await_approval_tool` (50 s per
  call, max 100 calls) while a human reviews the output at
  `/dashboard/reviews`. Approval resumes the pipeline hands-free; past the
  poll bound it reports `WAITING_APPROVAL` and stops.
- **approved with edits** — the reviewer can edit any `preserves-*` field in
  the dashboard before approving; edits are re-validated against the phase's
  SHACL gate (a violating edit changes nothing).
- **rejected** → the orchestrator re-dispatches the same phase agent with the
  reviewer's comment passed verbatim as `args.reviewer_feedback`; the agent's
  re-run replaces the rejected output (idempotent cleanup) and pends again.

Decisions are graph facts (`phase:approvalStatus`, `phase:reviewComment`,
`phase:reviewedBy`, `phase:reviewedAt`, `phase:editedByReviewer`) — auditable
like everything else. Approve/reject is deliberately **not** exposed over MCP,
so no agent can approve its own output. Outputs recorded before HITL existed
have no status triple and count as approved.

## Setup

### API key

The ontology-server requires a Bearer token. Export it before running:

```bash
export ONTOLOGY_API_KEY=$(cat ~/.ontology-server-key)
```

> **Note**: the key file is `~/.ontology-server-key` (hyphen), auto-generated by the ontology-server on first start. If you haven't started the server yet, do so once to create the key, then export it.

## Prerequisites

### 1. Start the ontology server

```bash
cd /path/to/semantic-tool-use
uv run python -m ontology_server
# or
uvicorn src.ontology_server.api.app:app --port 8000
```

### 2. Create an idea in the knowledge graph

Ideas must already exist in the knowledge graph before Tulla can process them.
Use the ontology server's `create_idea` MCP tool.
The idea must have at least a title and a `seed` or `backlog` lifecycle state.

## Launching the orchestrator

```bash
# From this repo's root
omnigent run . --prompt "Process idea-6"
```

Or pass the full URI:
```bash
omnigent run . --prompt "Run the pipeline for http://semantic-tool-use.org/ideas/idea-6"
```

Or run a single sub-phase:
```bash
omnigent run . --prompt "Run P3 for idea-6"
```


The orchestrator queries the phases graph, determines where the idea is in the
pipeline, and dispatches the appropriate sub-agent. If the pipeline was partially
run before, it resumes from where it left off.

## Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `ONTOLOGY_API_URL` | `http://localhost:8000` | Base URL of the ontology server MCP endpoint |
| `ONTOLOGY_API_KEY` | *(required)* | Bearer token for the ontology server — see [Setup](#setup) |

## Phase trace chain

Each sub-phase writes its output to:
```
http://tulla.dev/phase#<idea-id>-<phase-id>
```

with a `trace:tracesTo` pointer to the previous sub-phase URI:

```
idea-6 ← d1 ← d2 ← d3 ← d4 ← d5 ← [r1 ← r2 ← r3 ← r4 ← r5 ← r6 ←] p1 ← p2 ← p3 ← p4 ← p5 ← lw-trace
```

The orchestrator traverses this chain via `trace:tracesTo*` to query all
upstream facts in a single SPARQL query.



## Tool scoping (context-cost control)

The ontology MCP server advertises **63 tools**. Mounting all of them in every
agent injects ~10-20k tokens of unused tool schemas into each agent's context on
every turn — multiplied across the 19-agent fleet — and degrades tool-selection
accuracy.

Each agent therefore declares an MCP **allow-list** via `tools:` on its
`ontology:` block. Omnigent filters tool schemas at discovery time
(`runner/mcp_manager.py` → `_mcp_tool_schema`: `if allowed is not None and
bare_name not in allowed: return None`), so non-listed schemas are never sent to
the model — this is a real token saving, not a runtime guard.

```yaml
tools:
  ontology:
    type: mcp
    tools: [collect_upstream_facts_tool, record_phase_result_tool]  # <- allow-list
    url: "${ONTOLOGY_API_URL:-http://localhost:8000}"
```


| Agent | Allowed ontology tools |
|-------|------------------------|
| orchestrator (`config.yaml`) | `sparql_query` |
| D1 | `collect_upstream_facts_tool`, `record_phase_result_tool`, `get_idea`, `query_ideas` |
| D2, D3, P1, P2, P5, R1-R6 | `collect_upstream_facts_tool`, `record_phase_result_tool` |
| D4, P3 | + `query_ontology` |
| D5 | + `get_idea`, `append_to_idea` |
| P4 | + `store_facts` |
| P6 | + `recall_facts`, `store_facts` |
| I1_coding | + `recall_facts`, `recall_lessons`, `store_facts`, `forget_facts`, `set_lifecycle` |

The allow-list is orthogonal to the raw-write DENY policy below: the allow-list
controls *what schemas the model sees*; the policy is defence-in-depth for
*what may execute*.

## Work directory layout

Each pipeline run uses a consistent work directory `./work/<idea-id>/`:

```
./work/idea-6/
  d1_northstar.md
  d2_persona_exploration.md
  d3_scoring.md
  d4_gap_analysis.md
  d5_research_brief.md
  r1_question_refinement.md   (if research ran)
  r2_source_gathering.md      (if research ran)
  r3_investigation.md         (if research ran)
  r4_synthesis.md             (if research ran)
  r5_experiments.md           (if research ran)
  r6_recommendation.md        (if research ran)
  p1_requirements.md
  p2_architecture.md
  p3_design_decisions.md
  p4_implementation_plan.md
  p5_risk_assessment.md
```

## Polly orchestration (tulla-orchestrate skill)

The `tulla-orchestrate` skill at `skills/tulla-orchestrate/SKILL.md` is
the polly-facing interface. Load it when the user says "run tulla for idea-N"
or provides an idea URI. It contains the same routing table as the orchestrator
prompt but written for polly's `sys_session_send` dispatch style.

## Known notes

- **Count field datatypes**: All count fields use plain string literals (e.g.
  `"5"`). No typed `xsd:integer` literals — the SHACL shapes use `sh:minCount 1`
  only, with no `sh:datatype` constraint.
- **validate_by_uri response**: Returns `{"valid": <bool>, "violations": [...]}`.
  The key is `valid` (not `conforms`).
- **P5 blocking**: If `preserves-research_requests` from P5 is not `[]`, the
  orchestrator stops before dispatching I1 and surfaces the blocked status to
  the user.
