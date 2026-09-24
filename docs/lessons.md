# Lessons learned

These come from things that actually went wrong or nearly did, not from hindsight theory.

## A green check proves nothing until you have seen it fail

This was the lesson that kept coming back. Several fences shipped looking green and were blind:

- CI on the main branch was red for five merges in a row without anyone noticing the reason: a set of write-fence tests read a git-ignored file, so in CI they never ran at all.
- The first read-only posture eval gave the model no tools. Every check passed, because a model with no tools cannot state any fact, including a false claim that it can write.
- A formatter was rewriting the golden files it was supposed to leave alone, which meant the goldens were being updated to agree with the code.

What changed: every new fence is written and run red against the unfixed system first. Database harnesses include a control arm (so a fence that fails everything cannot look healthy) and inverted pairs (guard present versus absent, same fixture). CI discovers modules instead of reading a hand-typed list, because a hand-typed list produced a green run that had never looked at the newest module.

## Templates leak

For a period, every tenant was served the first tenant's system prompt. The prompt was a module constant written during the single-tenant phase, and nothing failed when a second tenant booted with it. The second tenant's agent booted with the first tenant's identity and capability list.

What changed: the prompt was split into an engine-owned safety spine and tenant-authored slots, with a boot check that every invariant is present. More generally, anything written during the single-tenant phase is now treated as suspect until a second tenant has exercised it.

## The UI can lie even when the engine is correct

A read-only tenant's dashboard said "Make changes, only with your OK" and listed write tools by name, because the screens read the tool config instead of the live wiring. The engine could not execute a single write, so nothing unsafe happened, but the product was telling the owner something false. The same bug meant that when the kill switch fired, the screens kept promising writes.

What changed: the UI and the prompt now derive what they advertise from the same state that gates execution. The test for it was written first and run red.

## Documentation drifts from production quietly

The egress documentation described a default-deny rule that production did not have for four days, because the rule was added to a script that nobody re-ran. It surfaced only because a later deploy asserted that the rules existed instead of assuming they did.

What changed: deploy scripts assert the live state they depend on, and a schema-drift check runs against the live business database before any grant change. That check immediately found a dormant write tool pointing at a column that had been dropped; the tool was withdrawn rather than patched in place.

## Multi-reviewer design pays for itself on outage-shaped bugs

Every non-trivial change had a written design reviewed separately from product, architecture, QA and AI-security angles, with conflicts ruled on in writing before code. For the readback-signature change, the three technical reviews independently rejected the specifics of my first draft, and two of the four problems they found would have caused a silent write outage on live tenants. The design phase is slower. It was cheaper than the incident.

## Keep the safety claims where the data is

The most useful architectural rule was putting the last fence in Postgres. Engine checks make for good error messages; database checks are what hold when the engine has a bug. The same thinking applied to identity (the tenant comes only from a verified token, never from a request field) and to secrets (never in the folder, so exports are clean by construction rather than by care).

## What I would do differently

- **Start with the second tenant in mind.** Most of the painful work was un-hardcoding assumptions from the first tenant: timezone, admin navigation, prompt identity. A throwaway second tenant from week one would have surfaced them earlier.
- **Put the tenant folder repos under the same review and backup as the main repo from day one.** For a while, a trusted-tier config file's only copy of record lived on one machine.
- **Measure retrieval before it matters.** The recall@k pack exists and encodes the expected drop-off, but query-aware retrieval is still future work. With larger memory stores it would have become the bottleneck.
