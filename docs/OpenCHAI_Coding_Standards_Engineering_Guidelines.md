# OpenCHAI Coding Standards & Engineering Guidelines

**Document 13 of the Phase 0 Architecture Series**
**Status:** Draft for review
**Depends on:** Docs 1–12 (Terminology through Repository Structure)
**Feeds into:** Five-Year Product Roadmap; day-to-day engineering practice from Phase 1 onward

---

## 1. Purpose and Scope

This is the last document before implementation begins. It sets the concrete, enforceable engineering practices — style, testing, error handling, logging, review process, and definition of done — that every contribution to the repository laid out in Doc 12 must follow. Where earlier documents defined *what* must be true architecturally (idempotency, tenancy isolation, the `spec`/`status` boundary), this document defines the day-to-day discipline that keeps those properties true as the codebase grows and more engineers touch it.

---

## 2. Language and Style

| Language | Formatter | Linter | Type checking |
|---|---|---|---|
| Python (api-server, controllers, adapters, workflow-engine) | `black` | `ruff` | `mypy`, strict mode on `resource-model/v1` and `controller-framework` (the modules everything else trusts, Repository Structure §4); standard mode elsewhere |
| TypeScript (web-ui, cli, sdk) | `prettier` | `eslint` | `tsc --strict` |

- Formatting is **never** a matter of reviewer opinion — `black`/`prettier` run in CI as a hard gate (Repository Structure §6's lint stage), and a PR failing formatting is rejected automatically before a human reviewer looks at it.
- `mypy`/`tsc --strict` on `resource-model/v1` specifically because every other module trusts its shapes (Repository Structure §4) — a type error there has the widest possible blast radius in the codebase.

---

## 3. Testing Standards

Extends the two testing strategies already specified in Doc 8 §9 (Controllers) and Doc 9 §9 (Adapters) into a uniform policy across the whole codebase:

| Test tier | Applies to | Requirement |
|---|---|---|
| **Unit** | Every module | Runs against fakes/mocks only (fake Adapter for Controllers, per Doc 8 §9; in-memory Resource Store for API Server service-layer logic); no network, no real database, no real target infrastructure. Must run in well under a second per test. |
| **Conformance** | Controllers, Adapters | Shared suite (Doc 8 §9, Doc 9 §9) — a new Controller/Adapter is not mergeable until it passes its conformance suite; this is a **required** CI gate (resolving Controller Framework Design §11 Q2 explicitly: required, not advisory). |
| **Integration** | Selected critical paths | Against real or realistically-simulated systems (Redfish simulator, a real local Slurm/K8s dev cluster); release-gating, not per-PR (Repository Structure §6), run on a schedule and before tagging a release. |
| **End-to-end** | Full-stack scenarios (e.g., the ClusterBringUp Workflow from Architecture Document §7.3 / Workflow Engine Design) | Run in a dedicated staging environment before promoting a release; validates the whole chain from API call to Adapter action and back, not just one component in isolation. |

**Every new Resource Kind, Controller, or Adapter must ship with tests demonstrating:**
1. The idempotency requirement (`apply()` called twice converges, per Adapter Interface Contract §9 item 1) — for Adapters specifically.
2. Correct `Retryable`/`Terminal` classification for at least one simulated failure of each type — for Adapters.
3. Correct `status.observedGeneration` handling under a mid-flight `spec` change (Reconciliation Model §2.2's stale-generation scenario) — for Controllers.

A PR introducing a new Controller/Adapter without these specific tests is incomplete by definition, not merely "could use more tests."

---

## 4. Error Handling Conventions

- **Adapters classify; nothing upstream re-classifies.** Once an Adapter returns `Retryable` or `Terminal` (Adapter Interface Contract §2.2), no Controller, Workflow Task, or API handler is permitted to override that classification based on its own guess from the exception type — if the classification is wrong, the fix belongs in the Adapter, not in a workaround further up the stack.
- **No bare `except:` clauses** anywhere in the Python codebase — every caught exception is either a specific, expected type (handled meaningfully) or re-raised. A silently swallowed unexpected exception is exactly the kind of bug that produces a mysteriously-stuck `Pending` resource with no trace of why.
- **User-facing errors always use the RFC 7807 envelope** (API Design Guidelines §13) with a stable `openchaiErrorCode` — a new error condition requires adding a documented code, not returning an ad hoc message string that a client (UI, CLI, SDK) would have to string-match.
- **Secrets never appear in an error message, log line, or stack trace.** Any exception raised from within credential-handling code (Security Model §6, §7; Adapter Interface Contract §6) must have its message sanitized before propagating — enforced by a lint rule scanning for credential-handle variables flowing into `str(exception)`/log calls in `adapters/*`.

---

## 5. Logging and Observability Conventions

Extends Controller Framework Design §8's structured logging requirement platform-wide:

- **Structured logging only** (JSON lines), never free-text `print`/unstructured log messages, across every deployable.
- Every log line touching a Resource includes, at minimum: `resource_id`, `kind`, `organization_id`, `project_id` (Security Model §5's tenancy fields, so logs can be scoped/redacted per tenant if ever needed), and — where applicable — `generation`/`observedGeneration` (Reconciliation Model §2.2) and `resourceVersion`.
- Log level discipline: `ERROR` is reserved for `Terminal` outcomes and unexpected exceptions (§4); `WARN` for `Retryable` failures and `Degraded`/`Blocked` conditions; `INFO` for phase transitions and successful reconciliations; `DEBUG` for per-attempt detail within a single reconciliation. This discipline exists specifically so `ERROR`-level alerting (Security Model §9, Domain Model §3.8's `Alert`) doesn't drown in routine retry noise.
- Metrics naming follows `openchai_<component>_<metric>` (e.g., `openchai_controller_reconcile_duration_seconds`), labeled consistently with the fields above, so dashboards built for one Controller type generalize to any other without bespoke queries.

---

## 6. Documentation Standards

- **Every public function/class in `resource-model/v1`, `controller-framework`, and `adapter-contract`** (the shared-trust modules, Repository Structure §4) requires a docstring stating its contract, not just its parameters — specifically: what it assumes about caller behavior (e.g., "must not be called concurrently for the same resource_id") and what it guarantees in return (e.g., idempotency).
- **Architecture Decision Records (ADRs)** are required for any decision that changes a rule established in Docs 1–12 of this series (e.g., choosing to adopt Temporal later, per Workflow Engine Design §2's "reconsider" note) — stored under `docs/architecture/adr/`, numbered sequentially, never silently superseding a Phase 0 document without a recorded reason.
- New terms introduced anywhere in the codebase or its docs must be added to the Terminology & Glossary (Doc 1 §9) in the same PR — this is a lint-adjacent review checklist item, not automatically enforced by tooling, since term detection isn't reliably automatable.

---

## 7. Git Workflow and Review

- **Trunk-based development**: short-lived feature branches, merged to `main` via PR, no long-lived release branches for the core platform (individual deployables are still tagged/released independently per Repository Structure §7, but from a single trunk).
- **Conventional Commits** (`feat:`, `fix:`, `refactor:`, `docs:`, `test:`) — required, since release tooling (Repository Structure §7's independent per-deployable versioning) derives changelogs and version bumps from commit type per affected module.
- **Required reviewers:** any PR touching `resource-model/v1`, `controller-framework`, `adapter-contract`, or `api-server/middleware` (the modules with the widest trust radius, Repository Structure §4) requires review from a second engineer with prior context in that specific module, not just any available reviewer — these are the places where a subtle mistake has the largest blast radius across the whole platform.
- **No direct pushes to `main`** under any circumstance, including for maintainers — the import-boundary and conformance CI gates (Repository Structure §6) exist specifically to catch violations of the architectural rules in Docs 1–12, and bypassing PR review bypasses them too.

---

## 8. Security Coding Practices

Restates and operationalizes Security & Multi-Tenancy Model rules as concrete coding rules:

- **Never construct a database query without going through the Repository Layer** (Repository Structure §4's rule that only `api-server` touches the database) — a Controller or Adapter reaching directly into PostgreSQL, even for a "quick read," is a hard rule violation, not a style preference, because it bypasses RLS enforcement paths that depend on the Repository Layer's session setup (Database Architecture §4).
- **Never log or persist a raw secret value** (§4 above) — code review must specifically check any new code path touching `Secret`/`Credential` resources against this rule; it is the single highest-severity category of review finding.
- **Every new endpoint must specify its required RBAC permission explicitly** (Security Model §4.2) in code — no endpoint is ever "implicitly" permitted by omission; a missing permission declaration is a merge-blocking CI check, not a runtime surprise.
- **Every new Controller/Task must re-validate AuthZ at execution time** where it acts on behalf of a user (Security Model §4.5, Workflow Engine Design §10) — this is checked in the conformance suite (§3), not left to individual code review to catch.

---

## 9. Dependency Management Policy

- New third-party dependencies (Python or TypeScript) require justification in the PR description: what it's for, why the standard library or an existing in-repo dependency doesn't suffice, and its license compatibility.
- Dependencies touching cryptography, secrets, or authentication (e.g., a JWT library, Vault client) require security review before adoption, given Security Model §3's token contract and §6's secret-handling requirements depend on these libraries behaving correctly.
- Dependency versions are pinned (lockfiles committed) for every deployable; automated dependency-update PRs (e.g., Dependabot-style) still go through the full CI gate (Repository Structure §6), including conformance suites — a dependency bump is not exempt from the same correctness bar as hand-written code.

---

## 10. Definition of Done

A change is not complete until:

1. It passes lint, type-check, unit, and (if applicable) conformance tests (§3).
2. It includes structured logging and metrics consistent with §5, if it introduces new reconciliation, Task, or API-handling logic.
3. It updates the Terminology & Glossary (§6) if new terms were introduced, and updates the relevant Phase 0 document (via ADR, §6) if it changes an established architectural rule rather than merely implementing one.
4. It has been reviewed by the required reviewer(s) per §7, with explicit sign-off on any security-sensitive code path per §8.
5. For anything touching a Resource Kind's schema: the `resource-model/v1` change and its ripple effects (API Server, affected Controllers/Adapters, Web UI codegen) land together, per Repository Structure §7's coordinated-release rule — never a partial, out-of-sync update.

---

## 11. Open Questions Carried Forward

1. Should the conformance suite requirement (§3) extend retroactively to Controllers/Adapters built during early Phase 3 experimentation before this document existed, or apply only prospectively? Recommend retroactive application before Phase 3 exits to a stable release, to avoid two tiers of code quality persisting long-term.
2. Automated glossary-term detection (§6) — flagged as not reliably automatable now; worth revisiting if a good tool/heuristic emerges, to reduce reliance on manual review diligence.
3. Security review process for new dependencies (§9) — needs a named owner/process (a security champion rotation? a fixed reviewer group?) rather than the general "requires security review" statement here.

---

*End of Coding Standards & Engineering Guidelines. This completes Documents 10–13 (Workflow Engine Design, Database Architecture, Repository Structure, Coding Standards & Engineering Guidelines). Next in sequence: Five-Year Product Roadmap.*
