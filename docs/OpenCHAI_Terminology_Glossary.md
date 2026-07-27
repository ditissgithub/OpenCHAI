# OpenCHAI Terminology & Glossary

**Document 1 of the Phase 0 Architecture Series**
**Status:** Draft for review
**Purpose:** Establish one canonical vocabulary for OpenCHAI so every later document — Resource Model, API Guidelines, Controller Framework, Workflow Engine, Database Architecture, Security — uses the same words to mean the same things.

> **Note on sequencing:** this document is being written after the Architecture Document and Resource Model Specification (Docs 2 and 3) rather than before them, since terms are easiest to define precisely once they've been used in context. Definitions below are retrofitted to match how those documents actually used each term; where a document used a term loosely, this glossary is the tie-breaker going forward.

---

## 1. How to Use This Glossary

- Every term here has exactly **one** definition. If a later document needs a term to mean something different, that is a signal to either rename the concept or update this glossary — not to let two meanings coexist silently.
- Terms are grouped by domain, then listed alphabetically within each group.
- Cross-references use → to point to the defining or most relevant entry.
- This glossary defines *concepts*, not field-level schema (see the Resource Model Specification for exact `spec`/`status` fields).

---

## 2. Core Concepts

| Term | Definition |
|---|---|
| **Control Plane** | The layer of OpenCHAI responsible for deciding and tracking *what should exist and in what state* — as distinct from the automation tools that carry out those changes. OpenCHAI *is* the Control Plane; xCAT/Ansible/Redfish/Slurm/Kubernetes are not. |
| **Resource** | The single unit of representation in OpenCHAI. Everything the platform manages or tracks (a Cluster, a Node, a GPU, a User, a Workflow...) is modeled as a Resource with the common schema defined in the → Resource Model Specification. |
| **Resource Kind** (or **Kind**) | The *type* of a Resource (e.g., `Cluster`, `Node`, `GPU`, `Workflow`). Analogous to a class, of which individual Resources are instances. |
| **Desired State** | The state a user or system has declared it *wants* a Resource to be in. Stored exclusively in a Resource's `spec` field. Never inferred — always explicitly authored. |
| **Observed State** | The state a Resource is *actually* in, as last reported by an → Infrastructure Adapter. Stored in a Resource's `status` field. Never written by users directly. |
| **Reconciliation** | The continuous process of comparing Desired State to Observed State and taking action to close any gap. Performed by → Controllers. |
| **Drift** | A detected difference between Desired State and Observed State that arose *without* a corresponding change to `spec` — e.g., a GPU driver was manually downgraded outside OpenCHAI. Drift is what Reconciliation exists to correct. |
| **Single Source of Truth** | The principle that exactly one durable store (the → Resource Store) holds the authoritative Desired State for the entire platform. No other system or cache is authoritative. |
| **Idempotent Operation** | An operation that produces the same end result no matter how many times it is executed with the same input — required of every → Infrastructure Adapter action so that retries are always safe. |

---

## 3. Structural / Component Terms

| Term | Definition |
|---|---|
| **API Server** | The single component authorized to read and write the → Resource Store. Enforces → AuthN, → AuthZ, and schema validation on every request. All other components go through it rather than touching the database directly. |
| **Resource Store** | The durable, versioned persistence layer (PostgreSQL) holding every Resource's `spec`, `status`, `conditions`, and `history`. The platform's → Single Source of Truth. |
| **Desired State Engine** | The logic that computes the difference between a Resource's `spec` and its `status` — i.e., detects → Drift — and surfaces it to → Controllers. Does not itself execute changes. |
| **Controller** | A long-running, restartable process that owns → Reconciliation for one → Resource Kind (or a small, cohesive set of related Kinds). Reads Desired State, reads Observed State, and invokes → Infrastructure Adapters to close the gap. Never talks to physical or external systems directly. |
| **Controller Manager** | The supervisory component that schedules, health-checks, restarts, and (via → Leader Election) coordinates multiple → Controller instances. Contains no domain-specific reconciliation logic itself. |
| **Workflow Engine** | The component that orchestrates multi-step, potentially long-running processes spanning more than one → Controller (e.g., "bring up a Cluster" = network + provisioning + Kubernetes install). Distinct from Controllers, which handle steady-state single-Kind reconciliation rather than multi-step orchestration. |
| **Workflow** | A → Resource Kind representing one multi-step orchestrated process. Composed of an ordered/DAG structure of → Tasks. |
| **Task** | A single step within a → Workflow — the smallest unit the Workflow Engine schedules, retries, and can roll back independently. |
| **Event Bus** | The transport mechanism by which Resource changes and status updates are published so → Controllers and the → Workflow Engine can react to them, instead of polling. Transport only — not durable storage (that's the → Resource Store's job). |
| **Infrastructure Adapter** (or **Adapter**) | The only component permitted to communicate with a specific external or physical system (xCAT, Redfish, Slurm, Kubernetes, LDAP, storage/network backends). Implements a fixed interface (`apply`, `read`, `delete`, `healthCheck`) so Controllers never need to know which underlying technology they're driving. |
| **Leader Election** | The mechanism by which exactly one instance of a singleton-sensitive component (e.g., a given Controller shard) is active at a time across multiple replicas, to prevent duplicate or conflicting actions, with standbys ready to take over. |

---

## 4. Resource Model Terms

| Term | Definition |
|---|---|
| **`apiVersion`** | The group/version identifier (e.g., `openchai.io/v1`) on every Resource, enabling schema evolution without breaking existing stored data. |
| **`spec`** | The portion of a Resource holding → Desired State. User-authored (directly or via automation); read-only to → Controllers. |
| **`status`** | The portion of a Resource holding → Observed State, plus coarse lifecycle (`phase`) and reconciliation bookkeeping. Written only by Controllers, never by users. |
| **`conditions`** | A structured, typed list of fine-grained health/progress signals on a Resource (e.g., `Ready`, `Degraded`, `Progressing`), each independently updatable, supplementing the coarser `status.phase`. |
| **`phase`** | The coarse lifecycle state of a Resource: `Pending`, `Provisioning`, `Ready`, `Degraded`, `Failed`, or `Terminating`. See the Resource Model Specification's lifecycle state machine. |
| **`history`** | An append-only log of a Resource's significant state transitions over time, retained for audit purposes. |
| **`resourceVersion`** | An optimistic-concurrency token incremented on every write to a Resource, used to detect and reject conflicting concurrent updates. |
| **`generation`** | A counter incremented only when a Resource's `spec` changes (not on `status` updates); compared against `status.observedGeneration` so a Controller can tell whether it has reconciled the *latest* intent. |
| **Ownership** (`ownerReferences`) | A strong, lifecycle-binding relationship between Resources: deleting the owner cascades deletion to what it owns (e.g., deleting a Cluster deletes its Racks and Nodes). See Resource Model Specification §5. |
| **Reference** | A weak, pointer-style relationship between Resources (e.g., a Node referencing an Image by ID) that does **not** cascade on delete; deletion of a still-referenced Resource is blocked or the referencing Resource is marked `Degraded`, per policy. |
| **Organization** | The top-level tenancy scope in OpenCHAI; owns Projects, Users, and Roles. |
| **Project** | A tenancy scope nested under an Organization; the typical scope for quotas and most infrastructure Resources (Clusters, Storage, Networks). |

---

## 5. Infrastructure Domain Terms

| Term | Definition |
|---|---|
| **Cluster** | A → Resource Kind representing a logical grouping of compute infrastructure (Racks, Nodes) managed as a unit, typically bound to one Scheduler and Software Stack. |
| **Rack** | A → Resource Kind representing a physical rack of hardware, owned by a Cluster, owning Nodes. |
| **Node** | A → Resource Kind representing one physical (or virtual) compute host — the unit that gets bare-metal-provisioned, imaged, and joined to a Scheduler/Kubernetes. |
| **Bare-Metal Provisioning** | The process of taking a physical server from powered-off/unconfigured to running its target OS image, performed via the xCAT and Redfish → Infrastructure Adapters. |
| **GPU** | A → Resource Kind representing one GPU device owned by a Node, tracked separately from the Node itself so driver versions, utilization, and health can be reconciled independently. |
| **OSProfile** | A → Resource Kind describing a reusable OS configuration (packages, kernel parameters) that a Node references. |
| **Image** | A → Resource Kind representing a bootable OS/software image referenced by Nodes during provisioning. |
| **SoftwareStack** | A → Resource Kind describing a versioned set of HPC/AI software components (e.g., CUDA, MPI) applied at the Cluster level. |
| **Scheduler** | A → Resource Kind representing the job-scheduling system (Slurm or Kubernetes) bound to a Cluster. |
| **Network / Subnet / Switch** | → Resource Kinds representing, respectively, a logical network, an IP range within it, and a physical/managed switch — together modeling the networking layer a Cluster depends on. |
| **Storage** | A → Resource Kind representing a storage volume or filesystem backing, with a type and capacity, referenced by Clusters/Nodes. |

---

## 6. Security & Identity Terms

| Term | Definition |
|---|---|
| **AuthN (Authentication)** | The process of verifying *who* is making a request, currently via LDAP bind, with Keycloak planned for the future. Enforced exclusively in the → API Server. |
| **AuthZ (Authorization)** | The process of verifying *what* an authenticated identity is permitted to do, based on → RBAC. Enforced exclusively in the → API Server, immediately after AuthN. |
| **RBAC (Role-Based Access Control)** | The model by which permissions are granted to Users via Roles scoped to an Organization or Project, rather than assigned to individual Users directly. |
| **User** | A → Resource Kind representing an authenticated identity, sourced from LDAP (or a future identity provider). |
| **Role** | A → Resource Kind representing a named set of permissions that can be bound to Users within an Organization or Project. |
| **Policy** | A → Resource Kind representing a platform behavior rule — e.g., whether auto-remediation of Drift is permitted for a given scope, distinct from RBAC (which governs *who* can act, not *how the system itself* behaves). |
| **Secret / Credential** | → Resource Kinds representing sensitive values and system-to-system credentials respectively; the Resource Store holds only references (e.g., into Vault), never raw values. |
| **Certificate** | A → Resource Kind representing a managed TLS/X.509 certificate and its expiry tracking. |
| **Multi-Tenancy** | The platform's support for multiple Organizations/Projects to share the same OpenCHAI deployment with strict data and access isolation between them. |

---

## 7. Operational & Observability Terms

| Term | Definition |
|---|---|
| **Audit** | A → Resource Kind representing a system-generated, immutable record of a significant action (who did what, to what, when). Distinct from `history` (which lives on the affected Resource) in that Audit records are queryable independently and centrally. |
| **Alert** | A → Resource Kind representing a firing or resolved notification condition (e.g., a Cluster is `Degraded`), with severity and acknowledgment tracking. |
| **Event** | A → Resource Kind representing a discrete, timestamped occurrence involving another Resource (e.g., "Node.StatusChanged") — the durable record corresponding to what the → Event Bus transports in real time. |
| **Backup** | A → Resource Kind representing a scheduled backup job and its retention policy for a Project or Cluster's state. |
| **Degraded** | A → `phase`/`conditions` value meaning a Resource is functioning but below its fully desired state (e.g., 9 of 10 Nodes healthy) — distinct from `Failed`, which means reconciliation could not proceed at all. |
| **Rollback** | The → Workflow Engine's process of reversing previously-completed → Task steps in a Workflow after a later step fails unrecoverably, returning affected Resources to their prior state. |

---

## 8. Terms Explicitly Avoided (and Why)

To prevent ambiguity creeping back in through informal usage, these near-synonyms are **not** used interchangeably in OpenCHAI documents:

| Avoid using interchangeably with | Because |
|---|---|
| "Automation Framework" ≠ "Control Plane" | OpenCHAI is explicitly *not* the former (Master Plan, Core Philosophy) — Ansible/xCAT/etc. are automation; OpenCHAI decides and tracks, it does not itself script. |
| "Config" ≠ "Spec" | "Config" is ambiguous about whether it's desired or observed. Use → `spec` for desired state always. |
| "State" alone ≠ "Desired State" / "Observed State" | Always qualify which one is meant; unqualified "state" is a common source of bugs in reconciliation logic. |
| "Job" ≠ "Task" ≠ "Workflow" | "Job" is reserved for Scheduler-level concepts (Slurm/K8s jobs) submitted *through* a Cluster's Scheduler Resource; "Task" and "Workflow" are OpenCHAI's own orchestration units. Don't conflate a Slurm job with an OpenCHAI Task. |
| "Plugin" ≠ "Adapter" | The Master Plan's Ecosystem phase mentions "Plugins" as a future, broader extensibility mechanism; "Adapter" specifically means the fixed `apply/read/delete/healthCheck` interface defined for infrastructure integrations today. Don't use them as synonyms until/unless the Plugins design explicitly merges them. |

---

## 9. Glossary Maintenance

- Any new term introduced by a subsequent Phase 0 document (Domain Model, API Design Guidelines, Controller Framework Design, Workflow Engine Design, Database Architecture, Security & Multi-Tenancy Model) must be added here in the same pass, not left implicit.
- If a later document needs to redefine a term listed here, that document must explicitly say so and this glossary must be updated in the same change — silent redefinition across documents is what this glossary exists to prevent.

---

*End of Terminology & Glossary. This document underlies all others in the Phase 0 series (Architecture Document, Resource Model Specification, and those still to come).*
