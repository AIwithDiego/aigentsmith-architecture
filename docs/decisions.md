# Architecture decisions

Ten decisions that shaped the system, in the order they were made. Each has the context, the choice, what was rejected, and what it cost.

---

## ADR-01: One sealed unit per tenant, not a shared multi-tenant service

**Context.** The first version ran inside one business's own database: agent tables next to business tables. It worked for one tenant. A second tenant forced the question of where memory, conversations and settings should live.

**Decision.** Each tenant becomes a self-contained folder (briefing, memory notes, SQLite store, charter, connectors, identities) run by its own engine process in its own rootless container. A thin control plane handles only auth, membership, token minting and fleet status.

**Rejected.** A shared service with a `tenant_id` column everywhere. It is cheaper to run and easier to deploy, but isolation then depends on every query remembering a filter, and the most damaging failure (memory from one business surfacing in another's conversation) is one missed predicate away.

**Tradeoffs.** More moving parts per tenant, per-tenant deploys, and a fleet to operate instead of one app. A migration from the old layout had to pass a frozen eval gate before it was accepted. In return, no tenant process has a code path to another tenant's data, and a host-level check asserts that no tenant's OS user can read another's folder or secrets.

---

## ADR-02: Tools are declarative config, and there is no generic SQL tool

**Context.** The agent needs to read many shapes of business data. The fastest way to give it that is a "run SQL" tool.

**Decision.** Every tool is a config entry: relation from an allowlist, fixed columns, typed params, declared filters and joins. The engine builds parameterised SQL. Identifiers are validated at load; values are always bound; results are row-capped.

**Rejected.** A generic SQL tool, even read-only. It turns every prompt injection into a query-writing opportunity and makes the agent's reach impossible to review.

**Tradeoffs.** New questions sometimes need a new tool or a database view, which is slower than letting the model improvise. Complex aggregates moved into views, which put more logic in the database. The upside is that a tenant's full read surface is one reviewable file, and onboarding a new business is mostly config.

---

## ADR-03: Every write is propose, confirm, commit, with state in the business's database

**Context.** The agent must be able to change data, and it must never do so without the owner's explicit agreement.

**Decision.** Write tools only stage a pending row and render a card. Confirmation is deterministic code bound to one proposal by id and single-use nonce. Commit is one transaction in the business's own database, with an audit row. Pending writes and the audit log live next to the business data, not in the agent's store.

**Rejected.** "Always allow" settings, model-decided confirmation, and keeping pending state in the agent's own store. The last one would split one logical transaction across two databases.

**Tradeoffs.** Every write costs the owner a tap. Undo had to be designed for every write tool, and some operations (deleting a scheduled social post) cannot be undone, so they are marked terminal and the card says so. The invariant is simple to state and test: zero unconfirmed writes.

---

## ADR-04: The runtime gateway lives inside the engine, and the control plane signs its tokens

**Context.** The web dashboard needs to reach a tenant's agent for chat and memory editing.

**Decision.** The gateway runs in the same process as the engine (so there is one writer on the tenant store), binds only to a private address, and accepts only Ed25519 tokens signed by the control plane: short-lived, single-use, scoped to exact verbs and paths, with the tenant taken only from the verified claims. The tenant holds only the public key.

**Rejected.** A public runtime API, long-lived API keys per tenant, and the browser talking to the runtime directly.

**Tradeoffs.** A token is minted per request, and the control plane has to be up for web chat to work. A compromised tenant cannot mint tokens, and a compromised control plane is recovered by swapping one keypair.

---

## ADR-05: A capability that is off is a tool that does not exist

**Context.** Early prompts told the agent what it was not allowed to do. Prompt refusals are probabilistic.

**Decision.** The per-tenant charter is applied at tool load. If a capability is off, the tool is not in the tool list. The prompt enumerates what the agent can change as a closed list derived from the tools that actually loaded.

**Rejected.** Exposing the full tool set and relying on instructions to refuse.

**Tradeoffs.** Changing what an agent can do means changing config and restarting, not editing a sentence. That friction is intended: permission changes should be deliberate and reviewed.

---

## ADR-06: The fence lives in the database, not the engine

**Context.** The engine validates everything, but the engine is also the component most likely to have a bug.

**Decision.** Every write restriction is also enforced by Postgres: column-level grants, row-level policies for time and ownership rules, narrow `SECURITY DEFINER` functions where a policy would be too broad, zero privileges on tables owned by other systems, and an in-migration proof that aborts when it finds a forbidden grant. The engine's checks exist to give good messages; the database's checks exist to hold.

**Rejected.** Engine-only enforcement with a broadly privileged database role.

**Tradeoffs.** More SQL, more migrations and a harder review, and the fence proofs themselves need testing against hostile grants. In return, an engine bug produces a failed write, not a bad one.

---

## ADR-07: Deterministic oracles instead of an LLM judge

**Context.** Agent behaviour has to be regression-tested across prompt and model changes.

**Decision.** Edge-state copy is written as exact strings during design. Evals check replies with contains, not-contains, regex and ordering oracles. Safety cases gate at 100%.

**Rejected.** A judge model scoring replies.

**Tradeoffs.** Oracles are brittle to harmless rewording, and some qualities (tone, helpfulness) are not covered. They are cheap, repeatable and explainable, and a failure points at a specific sentence.

---

## ADR-08: A readback signature instead of a version column

**Context.** The first stale-write guard was a version column on the row being written. But cards render from views that join other tables. If a joined row changed between propose and confirm (a page published in the CMS, say), the owner confirmed a card that was no longer true, and the guard did not notice.

**Decision.** Hash the declared readback columns at propose time and re-check the hash inside the commit statement. If anything the card showed has changed, the update matches zero rows and the owner gets a fresh card.

**Rejected.** Extending version columns to every joined table.

**Tradeoffs.** The readback columns must be declared per tool, and the change had to land in three steps because a mistake in the first step would have silently disabled all writes on live tenants. Both mechanisms ran side by side during the transition, each with its own CI job.

---

## ADR-09: Modules use table prefixes, not Postgres schemas

**Context.** The module shelf packages CRM, jobs, staff, inventory, billing, media, CMS and social as installable units. The obvious boundary is one Postgres schema per module.

**Decision.** Modules use table prefixes, and the manifest (not the schema) is the contract.

**Rejected.** Per-module schemas. A module has to lift into an existing business whose tables we did not name and cannot move. A schema boundary would have been fiction on arrival, and the executor and drift tools assume one schema.

**Tradeoffs.** Prefixes are a convention, not an enforced boundary, so the manifest has to carry the real contract: relations, columns, money units, enums, functions, grants and a never-write list, all checked against the live catalogue and pinned by a surface hash.

---

## ADR-10: The engine does not depend on its hosting

**Context.** The fleet ran on dedicated hosts during the pilot.

**Decision.** The engine runs unchanged against a plain folder (no container), so demos and development run locally, and hosted service is re-provisioned by script to the same security floor.

**Tradeoffs.** Hosted and local can drift, so the provisioning scripts assert the live state they depend on.
