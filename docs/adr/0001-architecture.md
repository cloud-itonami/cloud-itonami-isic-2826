# ADR-0001: TexMachAdvisor ⊣ Textile, Apparel and Leather Machinery Plant Operations Governor architecture

## Status

Accepted. `cloud-itonami-isic-2826` promoted from `:spec` to
`:implemented` in the `kotoba-lang/industry` registry, following the
verified fresh-scaffold protocol established by prior actors in this
fleet.

## Context

`cloud-itonami-isic-2826` publishes an OSS blueprint for textile/
apparel/leather production-machinery (weaving looms, industrial
sewing machines, leather-cutting machines, knitting machines and
fabric-cutting machines) **plant operations coordination**
(production-batch product-type/no-load-run-speed/quantity/defect-rate
data logging, assembly/test-bench-equipment maintenance scheduling,
safety-concern flagging, and outbound product shipment coordination).
Like every actor in this fleet, the blueprint alone is not an
implementation: this ADR records the governed-actor architecture that
promotes it to real, tested code, following the same langgraph
StateGraph + independent Governor + Phase 0->3 rollout pattern
established across the cloud-itonami fleet.

The closest domain analog is `cloud-itonami-isic-2818` (Manufacture of
power-driven hand tools): both are back-office coordination actors for
a fixed manufacturing plant with electromechanically-assembled,
test-bench-verified finished-goods output and a real physical/worker
safety dimension, and both share the same four-op shape
(`:log-production-batch`/`:schedule-maintenance`/`:flag-safety-
concern`/`:coordinate-shipment`), the same two-entity verified/
registered gate structure (equipment for maintenance scheduling, batch
for shipment coordination), and the same permanent equipment-actuation
and certification-authority blocks. This build mirrors
`cloud-itonami-isic-2818`'s architecture closely but adapts the hazard
profile, equipment vocabulary, and product taxonomy to the textile/
apparel/leather production-machinery plant: its finished goods are
production machinery for OTHER manufacturers (weaving looms,
industrial sewing machines, leather-cutting machines, knitting
machines, fabric-cutting machines) rather than hand-held consumer
power tools, so its equipment kinds are `:loom-assembly-line` and
`:sewing-machine-test-bench` rather than 2818's motor-assembly line
and housing-molding press, and its routine test-bench field is
`:no-load-run-speed-rpm` (plausibility-checked 0-8000 rpm, informed by
typical no-load/running-in test speeds across this vertical's own
equipment classes -- weaving looms approximately 200-800 rpm,
industrial sewing machines up to approximately 5,500 stitches/minute,
leather-cutting-machine blade-stroke rates typically below 1,000
strokes/minute -- and by the general mechanical-safety framework the
ISO 11111 series (Textile machinery -- Safety requirements)
establishes for this equipment class) rather than 2818's
`:hipot-test-kv` (electrical double-insulation withstand test,
plausibility-checked 0-15 kV against IEC 60745). Like 2818, shipment
quantity is tracked in finished-unit UNITS (`:units`/`:quantity-
units`/`:shipped-units`), since textile/apparel/leather production
machinery is likewise discrete counted units rather than a bulk
weight.

This vertical shares 2818's DOMAIN-SPECIFIC permanent block, adapted:
like power-driven hand tools, textile/apparel/leather production
machinery is subject to machinery safety certification regimes (e.g.
CE marking under the EU Machinery Directive 2006/42/EC, now Machinery
Regulation (EU) 2023/1230, informed generally by the ISO 11111 series'
safety-requirements framework for this equipment class). This actor is
never the certification authority -- any proposal (regardless of op)
that declares `:issue-certification? true` is a HARD, PERMANENT,
unconditional block
(`texmachmfg.governor/certification-authority-blocked-violations`),
the same "no phase, no human override" posture as the equipment-
actuation block.

This vertical has NO pre-existing `kotoba-lang/texmachmfg`-style
capability library to wrap (verified: no such repo exists). This build
therefore uses self-contained domain logic -- pure functions in
`texmachmfg.registry` (equipment/batch verification, shipment-
quantity recompute, product-type validation, no-load-run-speed
plausibility validation, defect-rate plausibility validation) are
re-verified independently by the governor, the same "ground truth, not
self-report" discipline established across prior actors (most
directly `cloud-itonami-isic-2818`'s `powertoolmfg.registry`).

This blueprint's own `:itonami.blueprint/governor` keyword,
`:textile-apparel-leather-machinery-plant-operations-governor`, is
grep-verified UNIQUE fleet-wide (`gh search code
"textile-apparel-leather-machinery-plant-operations-governor" --owner
cloud-itonami`, zero hits before this repo was created).

## Decision

### Decision 1: Self-contained domain logic (no external textile/apparel/leather-machinery-manufacturing capability library to wrap)

Unlike actors that delegate to pre-existing domain libraries, this
textile/apparel/leather production-machinery vertical has NO
pre-existing capability library to wrap. The equipment/batch-
verification / shipment-quantity / product-type / no-load-run-speed /
defect-rate validation functions live as pure functions in
`texmachmfg.registry` and are re-verified independently by
`texmachmfg.governor` -- the same "ground truth, not self-report"
discipline established across prior actors (most directly
`cloud-itonami-isic-2818`'s `powertoolmfg.registry`).

### Decision 2: Coordination, not control — scope boundary at the back-office

This actor is **strictly back-office coordination** of textile/
apparel/leather production-machinery plant operations. It does NOT:
- Control weaving-loom, sewing-machine, or leather-cutting-machine assembly/test-bench equipment directly
- Make plant-safety or certification decisions (exclusive to the human plant supervisor / accredited certification body)
- Actuate assembly/test-bench equipment
- Self-issue a machinery safety certification mark (e.g. CE marking under the EU Machinery Directive)

All proposals are `:effect :propose` only. The advisor proposes; the
governor validates; escalation paths funnel to human plant-supervisor
approval. This is not a replacement for the supervisor's authority or
the certification body's authority — it is a proposal-screening and
documentation layer.

**CRITICAL SAFETY BOUNDARY**: textile/apparel/leather production-
machinery manufacturing is a safety-critical domain (moving-part/
pinch-point hazard on assembly/test-bench lines, machinery safety
certification, downstream worker-safety consequence). Safety-concern
flagging NEVER auto-commits. All safety concerns escalate immediately
to human review.

### Decision 3: Safety-concern escalation — always human sign-off

`:flag-safety-concern` (mechanical-safety concern, electrical-safety
concern, CE-compliance concern, equipment-safety concern) ALWAYS
escalates, never auto-commits. This is not a "low-stakes proposal" --
it is a circuit-breaker that must reach human authority.

### Decision 4: Two independent verified/registered gates (equipment AND batch), not one

Like `cloud-itonami-isic-2818`, this vertical has TWO entity kinds
each gating a different op: `:schedule-maintenance` independently
verifies the referenced **equipment** unit's own `:verified?`/
`:registered?` fields; `:coordinate-shipment` independently verifies
the referenced **batch**'s own `:verified?`/`:registered?` fields.
Both are the same "plant/batch record must be independently
verified/registered before any action" HARD invariant applied to the
two distinct record kinds this domain actually has.
`:coordinate-shipment` additionally independently recomputes whether a
batch's own recorded shipped-to-date unit quantity plus the
proposal's own claimed unit quantity would exceed the batch's own
recorded production quantity -- never taken on the advisor's
self-report.

### Decision 5: HARD invariants (no override)

Four HARD governor invariants (elaborated into twelve concrete checks
in `texmachmfg.governor`, mirroring `cloud-itonami-isic-2818`'s own
elaboration of its HARD invariants into concrete checks) block
proposals and cannot be overridden by human approval:
1. Plant/batch record (equipment for maintenance, batch for shipment) must be independently verified/registered before any action is taken against it, and a shipment's quantity must independently recompute within the batch's own logged production quantity
2. Proposals must be `:effect :propose` only (never direct equipment control)
3. Direct assembly/test-bench-equipment control, equipment actuation, or self-issued machinery safety certification is permanently blocked
4. The op allowlist is closed — `:log-production-batch`/`:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` only

## Consequences

(+) Textile/apparel/leather production-machinery plant operations
back-office now has a documented, governed, auditable coordination
layer that funnels all decisions through independent validation
before human approval.

(+) The "coordination, not control" boundary is explicit in code: all
`:effect :propose`, all real-world actuation requires human plant-
supervisor sign-off, and no certification mark can ever be
self-issued.

(+) Scope is bounded and verifiable: four HARD invariants (elaborated
into twelve concrete governor checks) protect against scope creep into
unauthorized equipment operation, equipment actuation, or
certification self-issuance. Safety concerns are a circuit-breaker,
not a threshold.

(+) Safety-critical discipline is explicit: safety-concern flagging
cannot be rate-limited, suppressed, or auto-decided by phase gate.
Human review is mandatory.

(-) Still a simulation/proposal layer, not a real plant-operations
control system. Equipment actuation, line operation, and certification
issuance remain human-/institution-controlled via external channels.

(-) No integration with real plant-management databases (equipment
telemetry, batch tracking, freight dispatch, certification-body APIs)
— this is a standalone coordinator blueprint.

## Verification

- `cloud-itonami-isic-2826`: `clojure -M:test` green (all tests pass;
  see the superproject ADR and `kotoba-lang/industry` registry entry
  for the exact `Ran N tests containing M assertions, 0 failures, 0
  errors` output, verified from an independent fresh clone), `clojure
  -M:lint` clean, `clojure -M:dev:run` demo narrative exercises
  proposal submission, escalation, and every HARD-hold scenario
  directly (not-propose-effect, unknown-op, equipment-not-verified,
  batch-not-verified, shipment-quantity-exceeded, equipment-actuate-
  blocked, certification-authority-blocked, already-scheduled,
  invalid-product-type, invalid-no-load-run-speed, invalid-defect-
  rate).
- All source is `.cljc` (portable ClojureScript / JVM / nbb) — no
  JVM-only interop; the actor graph is invoked exclusively via
  `langgraph.graph/run*` (not `.invoke`, which is not cljs-portable).
- Audit ledger is append-only, all decisions are traced; every settled
  request (commit or hold) leaves exactly one ledger fact.
- `deps.edn` pins `io.github.kotoba-lang/langgraph` and
  `io.github.kotoba-lang/langchain` via `:local/root` directly in the
  top-level `:deps` (not only under a `:dev` alias), so a bare
  `clojure -M:test` resolves offline inside the monorepo checkout.
