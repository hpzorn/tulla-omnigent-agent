Process http://semantic-tool-use.org/ideas/idea-15 end-to-end (work dir ./work/idea-15).

This run is ALSO the E2E verification of the new SHACL gates. On top of your
normal orchestration loop, act as a read-only gate auditor. Do not skip phases,
do not work around failures — a fired gate is a finding, not an obstacle.

## Gate-audit duties (after EVERY sub-agent completion, before routing on)

1. Verify the phase actually persisted — a sub-agent claiming success without
   triples is a GATE-AUDIT FAILURE (stop, report):

   SELECT (COUNT(*) AS ?n) WHERE {
     GRAPH <http://semantic-tool-use.org/graphs/phases> {
       <http://tulla.dev/phase#idea-15-PHASEID> ?p ?o .
     }
   }

   (substitute PHASEID: d1, d2, ... p6, i1. Expect n > 0 after success.)

2. Verify the mechanical coverage fields at the checkpoints:
   - after P1: `phase:preserves-uncovered_mandatory_features` must be the literal "[]"
   - after P3: `phase:preserves-uncovered_features` must be "[]"
   - after P4: `phase:preserves-uncovered_features` must be "[]"
   - after P6: `phase:preserves-coverage_gate` must be "pass" AND
     `phase:preserves-uncovered_features` must be "[]"

   SELECT ?p ?v WHERE {
     GRAPH <http://semantic-tool-use.org/graphs/phases> {
       <http://tulla.dev/phase#idea-15-PHASEID> ?p ?v .
       FILTER(STRSTARTS(STR(?p), "http://tulla.dev/phase#preserves-"))
     }
   }

   Apply your routing table's blocking rows mechanically: any non-"[]"
   uncovered list or coverage_gate="fail" → BLOCKED, surface it, stop.

3. After P6, additionally count the coverage links written to Agent Memory
   (expect ≥ 4 — one per feature at minimum):

   SELECT (COUNT(?f) AS ?n) WHERE {
     GRAPH <http://semantic-tool-use.org/graphs/memory> {
       ?f ?p "prd:coversFeature" .
     }
   }

4. Track per phase for the final report: phase id | agent-reported ok |
   SHACL violations quoted by the agent (if it needed retries) | retry count |
   persisted triple count | coverage-field values.

## Expected server behavior (so you can tell signal from noise)

Every record_phase_result_tool call is validated server-side against that
phase's SHACL shape; nonconforming output is ROLLED BACK (zero triples
persist). Sub-agents retry up to 3 times using the returned violations, then
return FAILED quoting them verbatim. A phase that fired its gate and then
passed on retry is a SUCCESS for this test — record the violations as
evidence the gate is live. A phase that exhausts retries: stop the pipeline
and surface every violation verbatim.

## Final report

Write ./work/idea-15/e2e-gate-report.md and summarize the same to me:

1. Phase table: phase | gate ok | retries | violations seen | triples persisted.
2. The coverage chain end-to-end: D5 mandatory_features (verbatim) → P1
   mandatory_feature_coverage mapping → P4 feature_coverage → P6
   coverage_gate + the prd:coversFeature count from duty 3.
3. Overall verdict line: `E2E-GATES: PASS` only if every completed phase
   persisted validated output AND every coverage checkpoint was "[]"/"pass";
   otherwise `E2E-GATES: FAIL — <reason>`.

Notes: decide D5 mode on merit — the idea is deliberately well-specified, so
research is likely unnecessary; mode=plan or implement is expected. If P5
blocks with research_requests or any coverage row blocks, that IS correct
gate behavior: report it as such and stop rather than forcing progress.
