# OpenCHAI Infrastructure Control Plane — Architecture Document

**Document 2 of the Phase 0 Architecture Series**
**Status:** Draft for review
**Scope:** System architecture, component design, control/data flow, sequence diagrams, and end-to-end flowcharts for the OpenCHAI Control Plane.

---

## 1. Purpose and Scope

This document defines the architecture of OpenCHAI: how its components are organized, how they communicate, and how a request moves from a user action to a change in physical or virtual HPC/AI infrastructure and back.

It does **not** define the detailed Resource schema, API contract, or Controller Framework internals — those are covered in dedicated documents later in the Phase 0 series (Resource Model Specification, API Design Guidelines, Controller Framework Design). This document is the connective tissue that those documents will plug into.

**Read this document to understand:**
- What the major subsystems are and what each owns
- How a user-initiated change becomes a reconciled, running piece of infrastructure
- How the system behaves under drift, failure, and retry
- How the pieces map onto the technology stack and deployment topology

---

## 2. Architectural Principles (Recap and Application)

The Master Plan established these principles. Here we state how each one shapes concrete architecture decisions in this document:

| Principle | Architectural Consequence |
|---|---|
| Resource-oriented architecture | Every noun in the system (Cluster, Node, GPU, Workflow...) is a Resource with a uniform schema, stored in one place, addressable by one API pattern |
| Desired state management | The Resource Store holds *intent*; a separate path holds *observation*. These are never merged into one mutable object |
| Continuous reconciliation | Controllers run as independent, restartable loops — never as one-shot scripts triggered by a UI click |
| Event-driven workflows | State changes emit events; workflows and controllers react to events rather than polling business logic |
| API-first development | The API Server is the only writer to the Resource Store; no component reaches into the database directly |
| Modular controllers | One controller owns one Resource Kind (or a small, cohesive set); controllers do not call each other directly |
| Separation of control plane and automation | Ansible/xCAT/Redfish/Slurm/K8s are invoked *only* from Infrastructure Adapters, never from controllers or the API layer |
| Idempotent operations | Every adapter operation must be safe to retry; controllers assume at-least-once delivery of work |
| Scalability & HA | Stateless API/controller/worker processes; all durable state lives in PostgreSQL (+ future Redis/event bus) |
| Extensibility via adapters | New infrastructure integrations are new adapters implementing a fixed interface — core code does not change |

---

## 3. System Context

Before looking inside OpenCHAI, this diagram shows OpenCHAI as a single box among the actors and external systems it touches.

```mermaid
C4Context
    title OpenCHAI System Context

    Person(admin, "Infrastructure Admin", "Manages clusters, policies, users")
    Person(operator, "HPC/AI Operator", "Submits jobs, monitors clusters, requests resources")
    Person(developer, "Platform Engineer", "Integrates via API/SDK/CLI")

    System(openchai, "OpenCHAI Control Plane", "Resource-oriented control plane for HPC/AI infrastructure")

    System_Ext(hw, "Physical Infrastructure", "Servers, switches, GPUs, storage arrays, BMCs (Redfish)")
    System_Ext(k8s, "Kubernetes Clusters", "Container orchestration for AI workloads")
    System_Ext(slurm, "Slurm Clusters", "HPC batch scheduling")
    System_Ext(idp, "LDAP / Identity Provider", "Authn source, future Keycloak")
    System_Ext(monitoring, "Monitoring Stack", "Prometheus, Grafana, Loki")
    System_Ext(secrets, "Secrets Manager", "Vault (future)")

    Rel(admin, openchai, "Configures clusters, policies, RBAC", "Web UI / CLI / REST")
    Rel(operator, openchai, "Requests resources, views status", "Web UI / CLI / REST")
    Rel(developer, openchai, "Automates via", "REST API / SDK")

    Rel(openchai, hw, "Provisions, powers, images", "Redfish / xCAT")
    Rel(openchai, k8s, "Deploys, manages workloads", "K8s API")
    Rel(openchai, slurm, "Submits/manages jobs, nodes", "Slurm REST/CLI")
    Rel(openchai, idp, "Authenticates users", "LDAP bind")
    Rel(openchai, monitoring, "Pushes metrics/logs, pulls alerts", "Prometheus remote write / API")
    Rel(openchai, secrets, "Reads/writes credentials", "Vault API")
```

**Key takeaway:** OpenCHAI is the single point of intent for infrastructure. Every external system (Slurm, Kubernetes, Redfish-managed hardware, LDAP) is treated as a managed target, never as a peer that OpenCHAI defers decisions to.

---

## 4. High-Level Component Architecture

This expands the "Instead of / Architecture should follow" diagram from the Master Plan into a full component view.

```mermaid
flowchart TB
    subgraph Client Layer
        UI["React Web UI"]
        CLI["CLI"]
        SDK["SDK / 3rd-party clients"]
    end

    subgraph API Layer
        GW["API Gateway / Ingress"]
        AUTHN["Authentication Middleware"]
        AUTHZ["Authorization (RBAC) Middleware"]
        VALID["Schema Validation"]
        APISRV["OpenCHAI API Server (FastAPI)"]
    end

    subgraph Core Control Plane
        STORE[("Resource Store — PostgreSQL\n(Source of Truth)")]
        DSENGINE["Desired State Engine"]
        CTRLMGR["Controller Manager"]
        WFENGINE["Workflow Engine"]
        BUS(["Event Bus"])
    end

    subgraph Controllers
        C1["Provisioning Controller"]
        C2["Network Controller"]
        C3["GPU Controller"]
        C4["Storage Controller"]
        C5["Kubernetes Controller"]
        C6["Slurm Controller"]
        C7["Monitoring Controller"]
    end

    subgraph Infrastructure Adapters
        A1["xCAT Adapter"]
        A2["Redfish Adapter"]
        A3["Slurm Adapter"]
        A4["Kubernetes Adapter"]
        A5["LDAP Adapter"]
        A6["Storage Adapter"]
        A7["Network Adapter"]
        A8["Monitoring Adapter"]
    end

    subgraph Physical / External Infrastructure
        INFRA["Bare metal, GPUs, switches,\nstorage, K8s, Slurm, LDAP"]
    end

    UI --> GW
    CLI --> GW
    SDK --> GW
    GW --> AUTHN --> AUTHZ --> VALID --> APISRV
    APISRV <--> STORE
    APISRV --> BUS

    STORE <--> DSENGINE
    DSENGINE --> CTRLMGR
    BUS --> CTRLMGR
    BUS --> WFENGINE
    WFENGINE --> STORE
    WFENGINE --> BUS

    CTRLMGR --> C1 & C2 & C3 & C4 & C5 & C6 & C7
    C1 --> A1 & A2
    C2 --> A7
    C3 --> A2
    C4 --> A6
    C5 --> A4
    C6 --> A3
    C7 --> A8
    APISRV --> A5

    A1 & A2 & A3 & A4 & A5 & A6 & A7 & A8 --> INFRA
    INFRA -.observed state.-> A1 & A2 & A3 & A4 & A6 & A7 & A8
    A1 & A2 & A3 & A4 & A6 & A7 & A8 -.status updates.-> BUS
```

### 4.1 Component Responsibilities

| Component | Responsibility | Does NOT do |
|---|---|---|
| **API Server** | CRUD on resources, auth enforcement, schema validation, emits events on write | Talk to infrastructure directly; make scheduling/reconciliation decisions |
| **Resource Store (PostgreSQL)** | Durable, versioned storage of desired state, observed state, status, history | Business logic; it is a passive store |
| **Desired State Engine** | Computes diffs between desired and observed state; exposes "what needs to change" | Execute changes itself |
| **Controller Manager** | Schedules, supervises, and restarts individual controllers; assigns work | Contain domain logic for any specific resource kind |
| **Controllers** (Provisioning, Network, GPU, Storage, K8s, Slurm, Monitoring) | Reconcile one Resource Kind's desired vs. observed state by invoking adapters | Talk to hardware/software directly (must go through adapters) |
| **Workflow Engine** | Orchestrates multi-step, long-running, retryable, rollback-capable processes that span multiple controllers | Own steady-state reconciliation (that's the controllers' job) |
| **Event Bus** | Publishes resource-change and status events to interested subscribers | Store durable state (it is transport, not source of truth) |
| **Infrastructure Adapters** | Translate controller intent into calls against a specific technology (xCAT, Redfish, Slurm, K8s, LDAP...) | Make decisions about *whether* to act — only *how* to act |

---

## 5. Layered / Logical View

Another useful lens on the same system — how requests move down through logical layers regardless of which physical component instance handles them.

```mermaid
flowchart TB
    L1["Presentation Layer\nReact UI · CLI · REST clients"]
    L2["API Layer\nREST API · AuthN · AuthZ · Validation"]
    L3["Resource Layer\nResource Store (PostgreSQL) · Desired State"]
    L4["Control Layer\nReconciliation Controllers · Controller Manager"]
    L5["Orchestration Layer\nWorkflow Engine · Event Bus"]
    L6["Adapter Layer\nxCAT · Redfish · Slurm · K8s · LDAP · Storage · Network · Monitoring"]
    L7["Infrastructure Layer\nPhysical & virtual HPC/AI infrastructure"]

    L1 --> L2 --> L3 --> L4 --> L5
    L4 --> L6
    L5 --> L6
    L6 --> L7
    L7 -.observed state feedback.-> L6 -.-> L3
```

This is the layer diagram the Master Plan sketched (`GUI → Resource API → Desired State → Controllers → Workflow Engine → Infrastructure → Observed State → Reconciliation`), made explicit with the orchestration layer running alongside — not strictly after — the control layer, since workflows and controllers both react to the same event stream.

---

## 6. Detailed Component Diagrams

### 6.1 API Server — Internal Structure

```mermaid
flowchart LR
    subgraph API Server
        direction TB
        R["Router\n(REST resource routes)"]
        M1["AuthN Middleware\n(session/token/LDAP)"]
        M2["AuthZ Middleware\n(RBAC policy check)"]
        M3["Validation Middleware\n(Pydantic schemas)"]
        SVC["Resource Service Layer\n(business rules per Kind)"]
        REPO["Repository Layer\n(SQLAlchemy)"]
        EVT["Event Emitter"]
    end

    R --> M1 --> M2 --> M3 --> SVC --> REPO
    SVC --> EVT
    REPO <--> DB[("PostgreSQL")]
    EVT --> BUS(["Event Bus"])
```

### 6.2 Controller Manager and a Single Controller

```mermaid
flowchart TB
    subgraph Controller Manager
        SCHED["Scheduler / Leader Election"]
        REG["Controller Registry"]
        HEALTH["Health & Restart Supervisor"]
    end

    subgraph "Example: GPU Controller"
        WATCH["Watch Loop\n(subscribes to GPU resource events)"]
        RECON["Reconcile Function\ndesired vs observed"]
        ACT["Action Dispatcher"]
        STATUS["Status Writer"]
    end

    SCHED --> REG --> WATCH
    HEALTH -.monitors.-> WATCH
    WATCH --> RECON --> ACT --> ADAPTER["Redfish / GPU Adapter"]
    ACT --> STATUS --> API["API Server\n(status subresource update)"]
    BUS(["Event Bus"]) --> WATCH
```

**Reconciliation loop contract (applies to every controller):**
1. Read desired state for the resources it owns.
2. Read last-known observed state.
3. Diff. If no diff, sleep/wait for next event.
4. If diff, call the appropriate adapter action(s).
5. Write status/conditions back through the API Server (never directly to the DB).
6. Requeue on failure with backoff; never crash-loop the whole controller on a single resource's failure.

### 6.3 Workflow Engine — Internal Structure

```mermaid
flowchart TB
    subgraph Workflow Engine
        INTAKE["Workflow Intake\n(from API or Event Bus)"]
        PLANNER["Step Planner\n(DAG of tasks)"]
        EXEC["Task Executor Pool"]
        RETRY["Retry / Backoff Manager"]
        ROLLBACK["Rollback Manager"]
        WFSTATE[("Workflow State\n(in Resource Store)")]
    end

    INTAKE --> PLANNER --> EXEC
    EXEC --> RETRY --> EXEC
    EXEC -->|task failure, unrecoverable| ROLLBACK
    EXEC --> WFSTATE
    ROLLBACK --> WFSTATE
    EXEC --> CTRLS["Controllers (via their APIs/events)"]
```

### 6.4 Infrastructure Adapter — Common Shape

Every adapter, regardless of target technology, implements the same interface so controllers remain adapter-agnostic:

```mermaid
classDiagram
    class InfrastructureAdapter {
        <<interface>>
        +apply(spec) ActionResult
        +read(resource_ref) ObservedState
        +delete(resource_ref) ActionResult
        +healthCheck() AdapterHealth
    }
    class XCATAdapter
    class RedfishAdapter
    class SlurmAdapter
    class KubernetesAdapter
    class LDAPAdapter
    class StorageAdapter
    class NetworkAdapter
    class MonitoringAdapter

    InfrastructureAdapter <|.. XCATAdapter
    InfrastructureAdapter <|.. RedfishAdapter
    InfrastructureAdapter <|.. SlurmAdapter
    InfrastructureAdapter <|.. KubernetesAdapter
    InfrastructureAdapter <|.. LDAPAdapter
    InfrastructureAdapter <|.. StorageAdapter
    InfrastructureAdapter <|.. NetworkAdapter
    InfrastructureAdapter <|.. MonitoringAdapter
```

This uniform interface is what makes "extensibility through adapters" real: adding support for a new hypervisor or scheduler means writing one new class, not touching the Controller Manager, Workflow Engine, or API Server.

---

## 7. Sequence Diagrams

### 7.1 End-to-End: Admin Provisions a New Bare-Metal Node

```mermaid
sequenceDiagram
    actor Admin
    participant UI as React UI
    participant API as API Server
    participant DB as Resource Store
    participant Bus as Event Bus
    participant CM as Controller Manager
    participant PC as Provisioning Controller
    participant XA as xCAT Adapter
    participant RA as Redfish Adapter
    participant HW as Physical Node

    Admin->>UI: Define Node resource (spec: hardware, image, role)
    UI->>API: POST /resources/nodes
    API->>API: AuthN + AuthZ + Validate
    API->>DB: Insert Node (desired state, status=Pending)
    API->>Bus: Emit "Node.Created"
    API-->>UI: 201 Created (Node id, status=Pending)

    Bus->>CM: Node.Created event
    CM->>PC: Dispatch reconciliation for Node
    PC->>DB: Read desired spec + current observed state
    PC->>RA: Power on / set boot mode (Redfish)
    RA->>HW: IPMI/Redfish command
    HW-->>RA: Ack
    PC->>XA: Trigger OS provisioning (xCAT)
    XA->>HW: PXE boot + image install
    HW-->>XA: Install progress / completion
    XA-->>PC: Provisioning complete
    PC->>API: PATCH /resources/nodes/{id}/status (Ready)
    API->>DB: Update status + conditions + history
    API->>Bus: Emit "Node.StatusChanged"
    Bus-->>UI: Push update (via subscription/poll)
    UI-->>Admin: Node shown as Ready
```

### 7.2 Continuous Reconciliation (Drift Correction)

```mermaid
sequenceDiagram
    participant HW as Physical/Virtual Infra
    participant AD as Adapter
    participant CT as Controller
    participant DSE as Desired State Engine
    participant DB as Resource Store
    participant API as API Server
    participant Bus as Event Bus

    loop Every reconciliation interval / on event
        CT->>DB: Read desired state (via API)
        CT->>AD: read() observed state
        AD->>HW: Query actual configuration
        HW-->>AD: Current state
        AD-->>CT: Observed state
        CT->>DSE: Diff(desired, observed)
        alt Drift detected
            DSE-->>CT: Delta (e.g., GPU driver version mismatch)
            CT->>AD: apply(corrective spec)
            AD->>HW: Enforce desired config
            HW-->>AD: Result
            AD-->>CT: ActionResult
            CT->>API: Update status/conditions
            API->>DB: Persist + record history
            API->>Bus: Emit "Resource.Reconciled"
        else No drift
            CT->>API: Update status (unchanged, lastChecked=now)
        end
    end
```

### 7.3 Workflow Execution with Retry and Rollback (Example: Cluster Bring-Up)

```mermaid
sequenceDiagram
    actor Admin
    participant API as API Server
    participant WF as Workflow Engine
    participant NC as Network Controller
    participant PC as Provisioning Controller
    participant KC as Kubernetes Controller
    participant DB as Resource Store

    Admin->>API: POST /workflows (kind: ClusterBringUp)
    API->>DB: Persist Workflow resource (status=Pending)
    API->>WF: Emit Workflow.Created
    WF->>WF: Plan DAG: [Network, Provision Nodes, Install K8s]

    WF->>NC: Step 1 — configure subnet/VLAN
    NC-->>WF: Success

    WF->>PC: Step 2 — provision N nodes
    PC-->>WF: 8/10 nodes succeeded, 2 failed
    WF->>WF: Retry failed nodes (backoff)
    PC-->>WF: 10/10 nodes succeeded

    WF->>KC: Step 3 — install Kubernetes
    KC-->>WF: Failure (unrecoverable — incompatible kernel)
    WF->>WF: Trigger Rollback Manager
    WF->>PC: Rollback — deprovision nodes from Step 2
    PC-->>WF: Rollback complete
    WF->>NC: Rollback — release subnet from Step 1
    NC-->>WF: Rollback complete
    WF->>API: Update Workflow status = Failed (rolled back)
    API->>DB: Persist final status + full step history
```

### 7.4 AuthN / AuthZ Flow

```mermaid
sequenceDiagram
    actor User
    participant UI as UI/CLI
    participant API as API Server
    participant LDAP as LDAP Adapter
    participant RBAC as AuthZ Engine
    participant DB as Resource Store

    User->>UI: Login (username/password)
    UI->>API: POST /auth/login
    API->>LDAP: Bind/validate credentials
    LDAP-->>API: Valid + group memberships
    API->>DB: Load User + Role bindings
    API-->>UI: Session token (scoped claims)

    User->>UI: Request: delete Cluster X
    UI->>API: DELETE /resources/clusters/{id} (token)
    API->>API: AuthN — validate token
    API->>RBAC: AuthZ — check Role permits delete on Cluster in this Project
    RBAC-->>API: Allow / Deny
    alt Allowed
        API->>DB: Soft-delete / mark for teardown
        API-->>UI: 202 Accepted
    else Denied
        API-->>UI: 403 Forbidden
    end
```

---

## 8. End-to-End Flowcharts

### 8.1 Full Lifecycle: From API Call to Running Infrastructure and Back

```mermaid
flowchart TD
    START(["User submits desired state\nvia UI/CLI/API"]) --> VALIDATE{"Valid schema\n& authorized?"}
    VALIDATE -- No --> REJECT["Return 4xx error"]
    VALIDATE -- Yes --> PERSIST["Persist desired state\nin Resource Store\n(status: Pending)"]
    PERSIST --> EMIT["Emit resource-created event"]
    EMIT --> NOTIFY{"Simple resource\nor multi-step workflow?"}

    NOTIFY -- Simple --> CTRL["Relevant Controller\npicks up reconciliation"]
    NOTIFY -- Multi-step --> WF["Workflow Engine\nplans DAG of steps"]
    WF --> CTRL

    CTRL --> DIFF{"Desired ≠ Observed?"}
    DIFF -- No --> DONE1["Mark status: Reconciled\n(no-op)"]
    DIFF -- Yes --> ADAPT["Invoke Infrastructure Adapter"]

    ADAPT --> EXEC["Adapter executes against\nphysical/external system"]
    EXEC --> RESULT{"Success?"}
    RESULT -- Yes --> UPDATE["Update status/conditions\n+ append history"]
    RESULT -- No --> RETRYCHECK{"Retries\nremaining?"}
    RETRYCHECK -- Yes --> BACKOFF["Backoff, requeue"] --> ADAPT
    RETRYCHECK -- No --> FAILSTATE["Mark status: Failed"]
    FAILSTATE --> ROLLBACKCHECK{"Part of a\nworkflow?"}
    ROLLBACKCHECK -- Yes --> ROLLBACK["Workflow Engine\nrolls back prior steps"]
    ROLLBACKCHECK -- No --> ALERT["Emit Alert / Audit event"]
    ROLLBACK --> ALERT

    UPDATE --> EMITSTATUS["Emit status-changed event"]
    EMITSTATUS --> UI_PUSH["UI/CLI/API reflect\nnew status"]
    DONE1 --> UI_PUSH
    ALERT --> UI_PUSH

    UI_PUSH --> LOOP(["Controller continues\nwatching for future drift"])
```

### 8.2 Drift Detection Loop (Standalone Detail)

```mermaid
flowchart TD
    A(["Controller wake:\ntimer tick OR event received"]) --> B["Fetch desired state\n(from Resource Store via API)"]
    B --> C["Fetch observed state\n(via Adapter.read)"]
    C --> D{"Diff empty?"}
    D -- Yes --> E["Update lastChecked timestamp\nNo action taken"]
    D -- No --> F["Classify drift:\nconfig / capacity / health"]
    F --> G{"Auto-remediate\npolicy allows?"}
    G -- Yes --> H["Adapter.apply(desired)"]
    H --> I{"Applied successfully?"}
    I -- Yes --> J["Status: Reconciled\nAppend history entry"]
    I -- No --> K["Status: Degraded\nEmit Alert"]
    G -- No --> L["Status: DriftDetected\n(manual approval required)\nEmit Alert"]
    E --> M(["Sleep / wait for next trigger"])
    J --> M
    K --> M
    L --> M
```

### 8.3 New Adapter Onboarding Flow (Extensibility in Practice)

```mermaid
flowchart LR
    A["New target system identified\n(e.g., new storage vendor)"] --> B["Implement InfrastructureAdapter\ninterface: apply/read/delete/healthCheck"]
    B --> C["Register adapter in\nAdapter Registry (config, no code change elsewhere)"]
    C --> D["Define/extend Resource Kind spec\nfields relevant to new adapter"]
    D --> E["Controller Manager auto-discovers\nadapter via registry"]
    E --> F["Existing controller (e.g., Storage Controller)\nroutes to new adapter based on resource.spec.provider"]
    F --> G["No changes required to:\nAPI Server, Controllers, Workflow Engine, UI"]
```

---

## 9. Deployment / Runtime Topology

```mermaid
flowchart TB
    subgraph "Load Balancer / Ingress"
        LB["L7 Load Balancer"]
    end

    subgraph "API Tier (stateless, horizontally scaled)"
        API1["API Server pod 1"]
        API2["API Server pod 2"]
        APIN["API Server pod N"]
    end

    subgraph "Control Tier (stateless, leader-elected where needed)"
        CM1["Controller Manager\n(active)"]
        CM2["Controller Manager\n(standby)"]
        WF1["Workflow Engine worker 1"]
        WF2["Workflow Engine worker N"]
    end

    subgraph "Messaging"
        BUS["Event Bus\n(NATS / RabbitMQ — future)"]
        CACHE["Redis — future\n(caching, locks, rate limits)"]
    end

    subgraph "Data Tier"
        PG_P[("PostgreSQL Primary")]
        PG_R[("PostgreSQL Replica(s)")]
    end

    subgraph "Observability"
        PROM["Prometheus"]
        GRAF["Grafana"]
        LOKI["Loki"]
    end

    LB --> API1 & API2 & APIN
    API1 & API2 & APIN --> PG_P
    API1 & API2 & APIN --> BUS
    PG_P --> PG_R

    BUS --> CM1
    CM1 -.failover.-> CM2
    BUS --> WF1 & WF2
    CM1 --> PG_P
    WF1 & WF2 --> PG_P
    CM1 & WF1 & WF2 --> CACHE

    API1 & API2 & APIN & CM1 & WF1 & WF2 --> PROM
    PROM --> GRAF
    API1 & API2 & APIN & CM1 & WF1 & WF2 --> LOKI
```

**HA notes:**
- API Server tier is fully stateless — scale horizontally behind the load balancer with no session affinity required (auth is token-based).
- Controller Manager uses leader election; only one active instance reconciles at a time per controller shard to avoid duplicate actions, with hot standbys.
- Workflow Engine workers are stateless executors; workflow state itself lives in PostgreSQL, so any worker can pick up a queued step.
- PostgreSQL is the only stateful component in Phase 1–2; replication/HA for it is planned explicitly in Phase 7 of the roadmap.

---

## 10. Technology Stack Mapping

| Architecture Component | Technology (per Master Plan) | Notes |
|---|---|---|
| Web UI | React, TypeScript, Tailwind CSS, Vite | Talks only to REST API |
| API Server | Python, FastAPI, Pydantic | Owns AuthN/AuthZ/Validation |
| Resource Store | PostgreSQL, SQLAlchemy, Alembic | Single source of truth |
| Event Bus | NATS or RabbitMQ (future) | Not required for Phase 1–2 MVP; can start with in-process pub/sub |
| Cache / Locks | Redis (future) | Used by Controller Manager for leader election, rate limiting |
| Controllers / Controller Manager | Python services | One process type, many controller instances/threads |
| Workflow Engine | Python service | Could adopt an existing workflow library (e.g., Temporal-style pattern) rather than building fully custom — a decision for the Workflow Engine Design doc |
| Adapters | xCAT, Redfish, Slurm, Kubernetes API, LDAP, Docker | Each isolated so one failing integration doesn't affect others |
| Secrets | Vault (future) | Referenced by adapters needing credentials, never stored in plaintext in the Resource Store |
| AuthN/AuthZ | LDAP now, Keycloak (future) | RBAC model detailed in Multi-Tenancy & Security doc |
| Monitoring | Prometheus, Grafana, Loki | Controllers/adapters emit metrics & structured logs |

---

## 11. Non-Functional Requirements Addressed by This Architecture

- **Scalability:** Stateless API/controller/worker tiers scale horizontally; only the database requires vertical/HA strategy.
- **High Availability:** Leader election for singleton-sensitive components (Controller Manager); no single point of failure in the request path once the DB is made HA (Phase 7).
- **Idempotency:** Enforced at the adapter interface contract (`apply()` must be safe to call repeatedly with the same spec).
- **Auditability:** Every resource carries `history` and `events`; every state-changing action flows through the API Server, giving one place to enforce audit logging.
- **Security:** All external-system credentials are isolated behind adapters; AuthZ is enforced centrally in the API layer, not duplicated per-controller.
- **Extensibility:** New infrastructure targets are additive (new adapters), not invasive (no core changes) — shown explicitly in §8.3.

---

## 12. Open Questions for Subsequent Phase 0 Documents

These are flagged here so they're not lost, but are intentionally deferred to their owning document:

1. **Resource Model Specification:** Exact schema fields, versioning scheme (`apiVersion`), and relationship/ownership rules between Resource Kinds (e.g., does deleting a Cluster cascade to its Nodes?).
2. **API Design Guidelines:** REST conventions, pagination, subresource patterns (`/status`, `/events`), API versioning policy.
3. **Controller Framework Design:** Concurrency model per controller, requeue/backoff parameters, how controllers claim work in a multi-replica Controller Manager.
4. **Workflow Engine Design:** Build vs. adopt an existing durable-execution framework; DAG definition format; how rollback steps are declared per task.
5. **Database Architecture:** Migration strategy, partitioning/retention for `history`/`events`/`audit` at national-supercomputing scale.
6. **Security & Multi-Tenancy Model:** RBAC role/permission granularity, Organization/Project isolation guarantees at the query layer.

---

*End of Architecture Document. Next in sequence: Resource Model Specification.*
