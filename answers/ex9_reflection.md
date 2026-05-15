# Ex9 — Reflection

## Q1 — Planner handoff decision

### Your answer

In my Ex7 run (session sess_4cd56fc590f5), the planner produced a
single subgoal in each round — sg_1 "find venue near haymarket for
12" in round 1, sg_1 "retry with larger venue after rejection" in
round 2 — both assigned_half: "loop". The planner never assigned a
subgoal to structured.

The handoff decision was made by the executor, not the planner.
The loop agent called `handoff_to_structured` as an explicit tool
call once it had assembled a candidate booking, with the reason
"loop half identified a candidate venue; passing to structured half
for confirmation under policy rules." That prose is the agent's
self-description, not a planner routing signal.

The structured half then ran the deterministic policy check
(party_too_large) and rejected the first proposal. The bridge
performed a reverse handoff: state changed from structured → loop
with rejection_reason "party_too_large" attached. The planner
replanned in round 2, scaling from party_size 12 to 6. The loop
executor found The Royal Oak, handed off again, and the structured
half committed the booking (BK-B7655866).

The design lesson: the structured half's policy runs in Python,
outside any LLM influence. The loop half cannot misroute the policy
check because there is no routing — the only path to commitment is
through `handoff_to_structured`, and commitment only happens if the
Python rules pass. The LLM's prose ("under policy rules") is
irrelevant to correctness; the architecture enforces the constraint.

### Citation

- sess_4cd56fc590f5/logs/tickets/tk_590ed57f/raw_output.json — round 1 executor: venue_search(Haymarket, 12) → 0 results, then handoff
- sess_4cd56fc590f5/logs/tickets/tk_3646bd05/raw_output.json — round 2 executor: venue_search(Old Town, 6) → 1 result, then handoff
- sess_4cd56fc590f5/logs/trace.jsonl:7 — structured → loop reverse handoff, reason: party_too_large
- sess_4cd56fc590f5/logs/trace.jsonl:14 — structured → complete after round 2
- sess_4cd56fc590f5/session.json — result: booking BK-B7655866, party_size 6, the_royal_oak

---

## Q2 — Dataflow integrity catch

### Your answer

In session sess_bd54d2f09e46 (ex5-edinburgh-research), verify_dataflow
was never called — and the result shows exactly why it is necessary.

Subgoal 1 (research) ran venue_search three times against Haymarket and
returned 0 results every time. get_weather was first called with a
nonsense date (2023-10-25), failed, then retried with 2026-04-24 — one
day off from the requested 2026-04-25 — and returned rainy, 11°C.
Because venue_search found nothing, calculate_cost was never called.

Subgoal 2 (publish) received no structured data from subgoal 1 and its
agent explicitly acknowledged in its reasoning that it had "no access
to prior interactions" and would have to "synthesize" event details. It
then fabricated every field passed to generate_flyer: venue "The Royal
Oak" (invented), date 2023-11-24 (three years wrong), time 18:30 (wrong),
weather Sunny (directly contradicts the rainy, 11°C result from subgoal 1),
catering "Three-course meal" (requested: bar_snacks). The ticket
raw_output.json nevertheless reported success: true.

Without verify_dataflow, "success" is a claim the downstream code made
about itself. The fabricated values were individually plausible — a human
skimming the flyer would not reach for a calendar to check 2023-11-24.
The integrity check's value is precisely that it compares against
_TOOL_CALL_LOG ground truth rather than plausibility, which the flyer
agent cannot consult.

The lesson from absence is sharper than the lesson from detection:
dataflow integrity must be a structural guarantee, not an optional step.
Once subgoal 2 is allowed to generate_flyer without first proving every
data point traces to a real tool return, fabrication is the default.

### Citation

- sessions/sess_bd54d2f09e46/logs/tickets/tk_bea3aafd/raw_output.json — agent explicitly states it fabricated event_details
- sessions/sess_bd54d2f09e46/logs/trace.jsonl — no verify_dataflow event present
- sessions/sess_bd54d2f09e46/workspace/flyer.html — template fields (venue_name, total_gbp, temperature_c) are empty despite agent claiming success

---

## Q3 — Removing one framework primitive

### Your answer

The first production failure I'd expect is the loop half handing off a
fabricated venue to the structured half after venue_search returns zero
results. In sess_4cd56fc590f5, ticket tk_590ed57f shows exactly this:
venue_search(Haymarket, party_size=12) → 0 results, then the very next
call is handoff_to_structured with venue_id "Haymarket Tap" — a pub
that never appeared in any search return. The structured half rejected
the proposal for party_too_large, so the fabrication went unnoticed.
Had the party been ≤8 people, the same handoff would have been accepted
and committed: a booking at a pub that was never confirmed available.

The primitive that would surface this is **manifest discipline**. Each
executor ticket's manifest records which tool call outputs the downstream
action may legally draw on. Enforced at the handoff boundary, manifest
discipline requires that every field in the handoff payload — venue_id,
date, time, deposit — traces back to a real tool return in that ticket's
manifest. In tk_590ed57f the manifest shows venue_search returning 0
results; no venue IDs were emitted, so "Haymarket Tap" has no provenance.
A manifest check at handoff time would block the call and report that
venue_id is ungrounded, before the structured half could act on it.

This is not a contrived edge case. It appears in the first round of my
only Ex7 session, triggered by an ordinary request (party of 12 in
Haymarket). The fabrication only failed to cause harm because a separate
rejection reason fired first. In production the masking condition could
disappear — a smaller party, or a looser structured half — and the
first real booking might commit against a closed or non-existent pub.

The structural fix manifest discipline would impose is clear: no handoff
is emitted until every field in the payload has a provenance chain back
through the manifest to a tool call return. The agent's context string
("chosen venue haymarket_tap") is not provenance; only a venue_search
entry that includes haymarket_tap qualifies.

### Citation

- sess_4cd56fc590f5/logs/tickets/tk_590ed57f/raw_output.json — venue_search returns 0 results, then handoff_to_structured fabricates "Haymarket Tap"
- sess_4cd56fc590f5/logs/trace.jsonl:3-5 — venue_search → 0 results → handoff, no manifest check event between them
- sess_4cd56fc590f5/ipc/handoff_to_structured.json — final committed handoff payload (round 2) for contrast; round 1 payload was never persisted after rejection
