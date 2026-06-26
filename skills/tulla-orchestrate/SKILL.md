---
name: tulla-orchestrate
description: >-
  Run the full Tulla lifecycle pipeline for an idea URI across 20 individual
  sub-phase agents: D1-D5 (discovery), R1-R6 (research, optional), P1-P6
  (planning), I-loop (implementation orchestrator) → per-requirement coding
  sub-agents. Uses list_pipeline_tool (per family: "design", "research",
  "planning") and next_phase_tool to determine phase order and routing
  dynamically through P6; implementation loop is driven internally by I1_coding.
---

# tulla-orchestrate

The `tulla-orchestrate` skill drives the complete Tulla pipeline for a given idea
URI. Load it when the user provides an idea URI and wants to run (or resume) the
pipeline.

## When to load this skill

- User says "run tulla for idea-6" or "process <idea URI>"
- User provides an idea URI and wants the full pipeline
- User wants to resume a pipeline that stopped partway through
- User wants to run a specific sub-phase only (e.g., "run P3 for idea-6")

## Pipeline sequence

```
D1 → D2 → D3 → D4 → D5
                       ↓ mode="research"
                      R1 → R2 → R3 → R4 → R5 → R6
                       ↓ mode="plan"/"implement" (or after R6)
                      P1 → P2 → P3 → P4 → P5
                                              ↓ research_requests=[]
                                             P6
                                              ↓
                                           I-loop
                                              ↓ (per requirement, internal loop)
                                           impl-req-{taskId} (claude_code)
                                              ↓ all Complete or Blocked
                                           lifecycle=completed
```

## Procedure

1. Parse the idea URI from the user's message. Accept:
   - Full URI: `http://semantic-tool-use.org/ideas/idea-6`
   - Short form: `idea-6` → expand to `http://semantic-tool-use.org/ideas/idea-6`
   Derive the `idea_id` (e.g., `"idea-6"`) from the URI.

2. Call `collect_upstream_facts_tool(idea_id="<idea-id>")` to determine which
   phases are already complete. The returned dict's keys are completed phase_ids.

3. Get the canonical phase order by calling list_pipeline_tool once per family:
   - `list_pipeline_tool(agent_family="design")`   → `["d1","d2","d3","d4","d5"]`
   - `list_pipeline_tool(agent_family="research")` → `["r1","r2","r3","r4","r5","r6"]`
   - `list_pipeline_tool(agent_family="planning")` → `["p1","p2","p3","p4","p5","p6"]`
   If any call returns `[]` or an error, **stop immediately** (see Error Handling).
   Note: `"tulla"` is NOT a valid agent_family. Always use the three family names above.

4. Determine the next phase to run:
   - If no phases are complete, start with `d1`.
   - Otherwise, identify the last completed phase and derive its family from the phase_id prefix:
     - starts with `"d"` → `agent_family="design"`
     - starts with `"r"` → `agent_family="research"`
     - starts with `"p"` → `agent_family="planning"`
   - Call: `next_phase_tool(agent_family=<family>, current_id="<last_phase>", verdict="<verdict_or_empty>")`
   - Returns `{"next_id": "<phase_id>"}` within the same family, or `{"next_id": "terminate"}` when that family is exhausted.
   - **Cross-family transitions** (triggered when next_phase_tool returns `"terminate"`):
     - After `d5`: read `mode` from D5's facts (see D5 mode routing below):
       - `mode="research"` → next phase is `r1`
       - `mode="plan"` or `mode="implement"` → next phase is `p1`
       - `mode="park"` → stop, surface to user, do not proceed
     - After `r6` → next phase is `p1`
     - After `p6` → dispatch `I1_coding` (step 7), then terminate

5. Look up the agent name for the returned `next_id` using the agent map below,
   then dispatch via `sys_session_send`. Do this in the SAME turn.

6. End your turn. Collect inbox results and loop until `next_id == "terminate"` or
   a terminal lifecycle state is reached.

7. After P6 completes, dispatch `I1_coding`. I1_coding drives the implementation
   loop internally — it dispatches one claude_code sub-agent per requirement and
   loops until all requirements are Complete or Blocked. You do NOT drive the
   per-requirement loop; I1_coding does.

## Agent map (phase_id → agent name)

| Phase ID  | Agent name              | Family         |
|-----------|-------------------------|----------------|
| `d1`      | `D1_northstar`          | discovery      |
| `d2`      | `D2_persona_exploration`| discovery      |
| `d3`      | `D3_scoring`            | discovery      |
| `d4`      | `D4_gap_analysis`       | discovery      |
| `d5`      | `D5_research_brief`     | discovery      |
| `r1`      | `R1_question_refinement`| research       |
| `r2`      | `R2_source_gathering`   | research       |
| `r3`      | `R3_investigation`      | research       |
| `r4`      | `R4_synthesis`          | research       |
| `r5`      | `R5_experiments`        | research       |
| `r6`      | `R6_recommendation`     | research       |
| `p1`      | `P1_requirements`       | planning       |
| `p2`      | `P2_architecture`       | planning       |
| `p3`      | `P3_design_decisions`   | planning       |
| `p4`      | `P4_implementation_plan`| planning       |
| `p5`      | `P5_risk_assessment`    | planning       |
| `p6`      | `P6_prd_export`         | planning       |
| `i-loop`  | `I1_coding`             | implementation |

## Role discipline

Each phase family is restricted to its own domain. Never dispatch an agent for
work outside its boundary.

| Family          | IS allowed to                                              | IS NOT allowed to                                              |
|-----------------|------------------------------------------------------------|----------------------------------------------------------------|
| **discovery**   | Analyze problem space, personas, value, gaps               | Design architecture, make tech choices, write code             |
| **research**    | Answer research questions, write experiment scripts        | Design production architecture, write production code          |
| **planning**    | Synthesize facts, design architecture, write task lists    | Implement features, write production code, modify source files |
| **P6 (export)** | Parse P4 output, insert prd:Requirement triples            | Design architecture, write code, make tech choices             |
| **impl-loop**   | Orchestrate requirement loop, dispatch coding sub-agents   | Write production code, design architecture, make tech choices  |
| **claude_code** | Implement exactly one requirement, run verification        | Implement other requirements, redesign architecture            |

## D5 mode routing

After D5 completes, `next_phase_tool(agent_family="design", current_id="d5")` returns
`"terminate"` (D5 is the last design phase). Apply the cross-family transition manually:
check the `mode` field in collect_upstream_facts_tool's D5 entry and set `next_phase` directly:
- `mode="research"` → next phase is `r1`
- `mode="plan"` or `mode="implement"` → next phase is `p1` (skip R-phases)
- `mode="park"` → stop, surface to user, do not proceed

### P5 gate logic

After P5 completes, check the `research_requests` field in collect_upstream_facts_tool's P5 entry:
- `research_requests == []` → proceed, dispatch P6_prd_export
- `research_requests != []` → stop, surface blocked status to user. Do NOT dispatch P6.

### P6 gate

After P6 completes successfully (record_phase_result_tool ok=true), dispatch I1_coding.
I1_coding drives the implementation loop internally. You only need to dispatch it once.

P6 uses `store_facts_bulk` to write all prd:Requirement facts in a single MCP call
(one call per idea, not one call per field). The returned `{"stored": N, "errors": []}`
confirms the export count before P6 proceeds to record_phase_result_tool.

## Dispatching a sub-phase agent (D1–P6)

```
sys_session_send(
  agent="<agent_name>",
  title="<phase_id>-<idea-id>",
  args={
    "idea_uri": "<IDEA_URI>",
    "work_dir": "./work/<idea-id>"
  }
)
```

## Dispatching I1_coding (implementation loop)

```
sys_session_send(
  agent="I1_coding",
  title="i-loop-<idea-id>",
  args={
    "idea_uri": "<IDEA_URI>",
    "work_dir": "./work/<idea-id>"
  }
)
```

I1_coding handles the internal loop:
1. Reads all `prd:Requirement` facts from Agent Memory: `recall_facts(predicate="rdf:type", context="prd-idea-{idea_id}")`, filters results for `object="prd:Requirement"`, then reads each requirement's properties with `recall_facts(subject=req_uri, context="prd-idea-{idea_id}")`
2. Sorts by dependency graph using `prd:dependsOn` values (topological order)
3. For each unblocked pending requirement:
   - Marks `prd:InProgress` via `forget_fact(subject=req_uri, predicate="prd:status", context="prd-idea-{idea_id}")` + `store_fact(subject=req_uri, predicate="prd:status", object="prd:InProgress", context="prd-idea-{idea_id}")`
   - Dispatches fresh `claude_code` sub-agent with requirement spec + architecture context
   - Marks `prd:Complete` or `prd:Blocked` based on result using the same forget_fact/store_fact pattern
4. Loops until no `prd:Pending` requirements remain unblocked
5. Records summary via `record_phase_result_tool(phase_id="i1")`
6. Calls `set_lifecycle(lifecycle="completed")` on success

Use a consistent work directory for the entire pipeline run.
Create it before the first dispatch:
```bash
mkdir -p ./work/<idea-id>
```

## Lifecycle terminal states

Stop and report to the user when lifecycle reaches:
- `completed` — success (i-loop summary present and valid)
- `invalidated` — idea was a dead end (D5 or R1 returned early-termination)
- `failed` — a phase failed validation after retries
- `parked` — D5 mode="park" or idea needs human decision
- `blocked` — P5 research_requests not empty; needs human input

## Resume behaviour

If phases are partially complete (e.g., `p5` but not `p6`), pick up from where
the pipeline left off. collect_upstream_facts_tool shows which phase_ids are done.
Pass the last completed phase_id and its verdict to next_phase_tool.

## Running a single sub-phase

If the user asks to run only a specific sub-phase (e.g., "run P3 for idea-6"),
verify its prerequisite phase is present in collect_upstream_facts_tool results,
then dispatch only that agent.

## Error handling

### Ontology / SPARQL infrastructure errors (hard stop)

If ANY of the following orchestration-level tool calls returns an error or raises:
`collect_upstream_facts_tool`, `list_pipeline_tool`, `next_phase_tool`,
`render_tools_tool`, `render_gates_tool`, `render_input_contract_tool`,
`render_output_contract_tool`, `render_methodology_tool`, `record_phase_result_tool`

→ **Stop immediately. Do NOT dispatch the next phase.**

Report to the user:
- Which tool failed
- The error message returned
- The idea URI and phase being processed

Do NOT fall back to generic MCP tools (e.g. `get_idea`, `query_ideas`,
`sparql_query`) to patch up missing data. A partially-rendered phase prompt
is worse than no prompt: the sub-agent will improvise silently and produce
results that appear valid but are built on missing context.

### Empty results from required renderers (hard stop)

After calling `render_tools_tool` or `render_gates_tool` for a phase, verify the
returned markdown is non-empty. An empty tool list or empty gate list for any
phase other than a manually skipped one means the ontology server could not load
the phase definition.

→ **Treat empty output from a required renderer as a FAILURE. Stop and report.**

### Phase sub-agent validation failures

If a phase sub-agent returns a validation failure or an error:
1. Surface the failure to the user with: phase name, subject URI,
   violated SHACL constraints, and the current field values.
2. Stop — do not retry silently.
3. Let the user decide whether to correct the data and re-invoke.

## Work directory convention

```
./work/<idea-id>/
  p6-prd-export.ttl           (PRD RDF export — input to implementation loop)
  p6-prd-summary.md           (human-readable PRD table + dependency graph)
  i-loop-summary.md           (implementation loop summary: completed/blocked reqs)
```

All intermediate phase results (D1–D5, R1–R6, P1–P5) are stored exclusively in the
ontology knowledge graph via record_phase_result_tool. Use collect_upstream_facts_tool
to retrieve them. Only P6 and I1 write terminal deliverable files to the work directory.
