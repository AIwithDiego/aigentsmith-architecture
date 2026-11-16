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

## No tool for a table means the agent answers from its briefing

An owner asked why the agent listed three service packages when the database held five. The obvious diagnosis was a stale briefing. The real cause: the read-only role could see the packages table, but no tool exposed it. Tools are deny-by-default, so database access is not agent access. With nowhere to look, the model answered from the tenant briefing, and because the briefing sits inside the system prompt, the model treated it as instruction rather than background. The rule "every name in an answer comes from a tool call" could not hold.

What changed: a reachability report that compares the live schema with the tools a tenant actually has, because the gap is invisible from inside the repo (tenant validation can only see the tools that exist). On that tenant, 25 of 38 readable tables were reachable and 13 were invisible to the agent. Another tenant on the same architecture was at 100%, so curated tools can get there; this one had drifted because nothing measured it. The report changes nothing by itself. Plenty of tables should stay unreachable, but each one is now a decision instead of an accident.

## Two halves of one audit row, made two different ways

Undo on a stock reorder point told the owner "the record changed after my change" when nothing had changed. The before-image was read back through the database (`18.00`); the after-image was the raw string the agent proposed (`20`); the column holds `20.00`. The cast happened server-side, so the bound string was not the string that came back, and the undo's comparison failed. It was found by driving the live demo tenant after a deploy, not by a test.

What changed: the audit records what the database holds, never what we asked it to hold. Every written column comes back through `RETURNING`, rendered the same way as the readback that produced the before-image. A string assertion on the SQL could not have caught this, because the SQL was right; the new test executes the statement and reads what comes back, with a control arm proving the two strings really differ.

## A kill switch with no key

The pause-writes switch on one tenant returned an error, because the credential it needed had never been provisioned on that box. The engine had logged a warning about it at every boot. The tenant that could publish irreversibly to a social account was the one without the switch; the tenant that had the credential could not write at all.

What changed: the writer role was created and the switch was verified end to end on the live gateway, with a probe that first proved it was talking to the build it meant to test. A warning in a boot log is not an alarm.

## A sensor that reported healthy while blind

The brute-force jail protecting the operator console reported ACTIVE with zero failures for over a week. The web server was writing into a deleted log file while the jail tailed a new, empty one. "Total failed: 0" is what a quiet week looks like, and also what a blind sensor looks like. The cause was a fix for a different problem: a deploy step that created the log file if it was missing used a command that replaces the file when it exists, so every deploy orphaned the writer. It surfaced while deploying something else, when a look at the web server's open files showed the deleted one.

What changed: the deploy step now only creates the file when it is absent. And the deploy no longer assumes: after reloading the web server it checks for a deleted log file, restarts if it finds one, and fails the deploy if that did not fix it. That check fired on its first run.

## The obvious repair would have broken production three ways

One tenant's whole write layer had disappeared: write tools declared, but no database functions and no writer role behind them. Writes failed closed, so nothing bad happened, but nothing worked either. The obvious fix was to replay the original migrations. Checked against production first, that replay would have dropped three values the live admin app depended on, reverted a corrected stock-direction rule, and left in place a row-level policy bug that made one write fail with permission denied.

What changed: the restore was written from the live function bodies, not from the old migration files, and it refuses to run if it finds a value it does not recognise.

## Documentation drifts, firewalls drift

A deploy failed with a connection timeout that looked like a network blip. The real cause: a host had been added to the build-time egress allowlist and applied to one box but not the other, because nobody re-runs a firewall script to pick up a changed string. The same drift had already happened once on the same file.

What changed: the deploy compares the box's allowlist with the repo's before it starts, names the missing hosts, and fails instead of fixing it, because a deploy script must not widen a box's egress on its own authority.

## What I would do differently

- **Start with the second tenant in mind.** Most of the painful work was un-hardcoding assumptions from the first tenant: timezone, admin navigation, prompt identity. A throwaway second tenant from week one would have surfaced them earlier.
- **Put the tenant folder repos under the same review and backup as the main repo from day one.** For a while, a trusted-tier config file's only copy of record lived on one machine.
- **Measure retrieval before it matters.** The recall@k pack exists and encodes the expected drop-off, but query-aware retrieval is still future work. With larger memory stores it would have become the bottleneck.
