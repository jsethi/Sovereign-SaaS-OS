Sovereign SaaS OS

A production architecture for running modern SaaS on open infrastructure without making your application dependent on a single cloud provider.

Sovereign SaaS OS is an open playbook for building a portable, recoverable, AI-ready SaaS platform using Kubernetes, PostgreSQL, S3-compatible object storage, GitOps, and independently verifiable disaster recovery.

The reference implementation runs on Hetzner Cloud + dedicated servers, but Hetzner is deliberately treated as an implementation detail—not an application dependency.

Compute is disposable. State is recoverable. Recovery is independently verifiable.

Read the full Sovereign SaaS Playbook →

⸻

Why This Exists

There are two common ways to build SaaS infrastructure.

At one extreme, you assemble open-source tools yourself and end up with a platform that works until a node fails, a database corrupts, a secret leaks, or the person who built it leaves.

At the other extreme, you compose dozens of proprietary managed cloud services and gain operational convenience at the cost of increasing infrastructure cost, architectural coupling, and migration difficulty.

Sovereign SaaS OS explores a third path.

Use mature open-source infrastructure, but apply the same operational discipline expected from serious managed platforms:

* explicit failure models;
* defined RPO and RTO targets;
* immutable and independently stored backups;
* declarative infrastructure;
* GitOps;
* least-privilege secrets;
* deterministic rebuilds;
* destructive recovery tests;
* portable application interfaces;
* cross-provider disaster recovery.

The goal is not to self-host everything.

The goal is to maintain credible exit paths.

⸻

What “Sovereign” Means

Sovereignty does not mean having no vendors.

It means your system can be:

rebuilt, recovered, verified, and relocated without requiring the continued cooperation of a single infrastructure provider.

An application built on Sovereign SaaS OS talks to portable interfaces:

HTTP
PostgreSQL
S3-compatible object storage
Redis
Kubernetes
environment-based configuration

It should not care whether the underlying implementation is Hetzner today, another Kubernetes environment tomorrow, or something else several years from now.

⸻

The Five Invariants

Every design decision in the playbook is evaluated against five requirements.

1. Rebuildability

Compute is ephemeral.

A destroyed environment should be reconstructible from:

Terraform
+ Talos configuration
+ Git
+ bootstrap secrets
+ durable state

2. Recoverability

Every stateful component has a defined:

failure mode
recovery mechanism
RPO
RTO
certification test

A backup that has never been restored is not considered verified.

3. Portability

Application code remains independent of Hetzner-specific APIs and infrastructure primitives.

Infrastructure may change substantially without requiring the application architecture or data model to be redesigned.

4. Operability

Bootstrap, failover, recovery, upgrades, secret rotation, and disaster recovery should be encoded as repeatable procedures rather than tribal knowledge.

5. Independence

A recovery mechanism cannot depend solely on the thing it exists to recover.

For example:

PostgreSQL in Kubernetes
        ↓
backup outside Kubernetes
MinIO on Hetzner
        ↓
immutable recovery copy outside MinIO
Hetzner recovery storage
        ↓
independent Tier-3 provider

⸻

Architecture

The reference architecture looks roughly like this:

                         INTERNET
                             │
                             ▼
                    Hetzner Load Balancer
                             │
                             ▼
                       Cilium Gateway
                             │
               ┌─────────────┴─────────────┐
               │                           │
               ▼                           ▼
            Next.js                    Node API
           stateless                  stateless
               │                           │
               └─────────────┬─────────────┘
                             │
                  Auth / Authorization
                     Tenant Context
                       Rate Limits
                             │
            ┌────────────────┼────────────────┐
            │                │                │
            ▼                ▼                ▼
        PgBouncer          Redis            MinIO
        CNPG Pooler      disposable       distributed
            │                               EC:2
            ▼                                 │
       PostgreSQL                              │
      3 instances                              │
            │                                  ▼
            │                        Immutable Snapshots
            ▼                              Tier-2
   Hetzner Object Storage                      │
     Base Backup + WAL                         │
            │                                  │
            └────────────────┬─────────────────┘
                             ▼
                   Independent Provider
                    e.g. Backblaze B2
                         Tier-3
                   COMPLIANCE retention

The Kubernetes substrate uses:

Talos Linux
upstream Kubernetes
Cilium
Argo CD
CloudNativePG
MinIO
Prometheus
Grafana
Loki
OpenTelemetry / Tempo

The example application stack uses:

Next.js
Node.js
Better Auth
PostgreSQL + pgvector
Redis
S3-compatible object storage

The principles do not require that exact application stack.

⸻

The Important Difference: Recovery Is Part of the Architecture

Most architecture diagrams stop here:

Application
    ↓
Database
    ↓
Storage

Sovereign SaaS OS keeps going:

Application
    ↓
Primary State
    ↓
External Recovery State
    ↓
Independent Recovery State
    ↓
Verified Restore

For PostgreSQL:

CNPG PostgreSQL
      │
      ├── base backups
      └── WAL archive
              │
              ▼
     Hetzner Object Storage
              │
              ▼
       Independent Tier-3

For application objects:

MinIO
  │
  ├── local erasure coding
  ├── local version history
  │
  ▼
immutable point-in-time snapshots
  │
  ▼
Hetzner Object Storage
  │
  ▼
independent compliance-locked Tier-3

These are deliberately different mechanisms.

Replication copies state. Versioning preserves history inside a storage system. Backup preserves independently recoverable states.

⸻

Machine-Verifiable Failure Contracts

This project does not define reliability as “the YAML looks correct.”

It defines expected behavior and then deliberately causes failures to see whether the system behaves that way.

Examples include:

Kill the PostgreSQL primary
→ Did the synchronous transaction survive?
Destroy the PostgreSQL cluster
→ Can PITR reconstruct it from external WAL?
Lose one MinIO node
→ Can the system still read and write?
Lose two MinIO nodes
→ Does the documented quorum behavior occur?
Kill a snapshot halfway through
→ Is the incomplete recovery point rejected?
Remove Hetzner recovery credentials
→ Can the system restore entirely from Tier-3?
Kill an API pod during traffic
→ Do in-flight requests drain safely?
Make PostgreSQL unreachable
→ Does readiness fail without causing a restart storm?
Attempt cross-tenant access
→ Do both application authorization and PostgreSQL RLS deny it?

The intended result is not merely infrastructure-as-code.

It is failure behavior as code.

⸻

Who This Is For

Sovereign SaaS OS is aimed at:

Technical founders building SaaS products who want serious infrastructure without immediately adopting a large hyperscaler footprint.

Small platform and engineering teams that want Kubernetes and open-source infrastructure but need a disciplined production operating model.

AI SaaS teams running PostgreSQL/pgvector, object storage, queues, APIs, RAG workloads, or background processing.

Organizations concerned about cloud concentration risk and wanting a realistic path to move infrastructure later.

Engineers who want to own their architecture without pretending operations are easy.

It is probably not the right starting point if you:

* have a tiny application that is well served by a PaaS;
* do not need Kubernetes;
* do not have the capability to operate stateful infrastructure;
* would prefer to outsource database, object-storage, and cluster operations entirely;
* value minimum operational responsibility above infrastructure control.

Sovereignty has an operational cost. The point of this architecture is to make that cost explicit and manageable, not pretend it does not exist.

⸻

Why Hetzner?

Hetzner is the reference implementation because it provides an attractive combination of:

cloud VMs
dedicated servers
private networking
load balancing
S3-compatible object storage
strong price/performance

But the architecture intentionally limits how much Hetzner leaks upward into the application.

The long-term portability objective is:

Application portability       Very High
Kubernetes workload portability High
Platform-service portability    Moderate–High
Infrastructure-code portability Lower

Moving providers can require substantial infrastructure work.

It should not require rewriting the product.

⸻

Repository Status

The repository currently contains the canonical architecture and operating playbook.

The next phase is turning those contracts into a complete reference implementation.

Planned implementation structure:

.
├── README.md
├── Sovereign_SaaS_Playbook.md
├── Makefile
│
├── terraform/
│   ├── hetzner-network/
│   ├── control-plane/
│   ├── workers/
│   ├── tier2/
│   └── tier3/
│
├── talos/
│   ├── controlplane/
│   ├── workers/
│   └── patches/
│
├── infra/
│   ├── argocd/
│   ├── cilium/
│   ├── hccm/
│   ├── database/
│   ├── minio/
│   ├── redis/
│   ├── monitoring/
│   └── app/
│
├── scripts/
│   ├── bootstrap.sh
│   ├── restore-postgres.sh
│   ├── restore-object-snapshot.sh
│   └── rotate-auth-secret.sh
│
├── tests/
│   ├── substrate-cert.sh
│   ├── db-cert.sh
│   ├── minio-cert.sh
│   ├── snapshots-cert.sh
│   ├── tier3-cert.sh
│   └── app-cert.sh
│
└── docs/
    ├── BOOTSTRAP.md
    ├── DATABASE_PITR.md
    ├── OBJECT_RECOVERY.md
    ├── CLUSTER_LOSS.md
    ├── PROVIDER_LOSS.md
    └── UPGRADE_POLICY.md

⸻

How to Use This Repository

If You’re Evaluating the Architecture

Start with this README, then read:

The Sovereign SaaS Playbook

The playbook contains the complete rationale, failure models, recovery architecture, component responsibilities, bootstrap ordering, and certification model.

Do not start by copying individual YAML fragments.

Start by understanding the invariants and failure boundaries.

If You’re Building a New SaaS Platform

Use the playbook as an architecture baseline.

Decide which contracts apply to your workload, then substitute implementation details where appropriate.

For example:

Hetzner       → another infrastructure provider
MinIO         → another S3-compatible object system
Better Auth   → another authentication system
Next.js       → another frontend/runtime
Node.js       → another API runtime

The five sovereign invariants should survive those substitutions.

If You’re Implementing the Reference Architecture

The intended bootstrap path is:

Terraform
    ↓
Talos
    ↓
Kubernetes
    ↓
Cilium + HCCM
    ↓
Argo CD
    ↓
Operators
    ↓
PostgreSQL
    ↓
MinIO
    ↓
Recovery pipelines
    ↓
Application
    ↓
Certification

The future goal is to reduce this to commands such as:

make bootstrap
make certify
make restore-tier2
make restore-tier3
make rebuild-staging

⸻

Canonical vs. Certified

These words have specific meanings in this project.

Canonical
The architecture or contract has been approved.

Certification-ready
Executable tests exist for the contract.

Certified
Those tests have actually passed against a specific deployed environment and version set.

Continuously certified
The tests are repeated on a defined schedule and their results are retained.

A YAML file existing in Git does not make infrastructure certified.

⸻

Production Readiness Philosophy

A healthy production system is not merely one that is currently serving HTTP requests.

Production readiness includes:

application healthy
database healthy
replication healthy
WAL archive current
object recovery point current
Tier-3 copy current
secrets manageable
network policy validated
restore procedures tested
failure contracts passing

A system whose application is online but whose recovery chain is broken is degraded.

⸻

Design Principles

A few rules appear repeatedly throughout the playbook:

Git defines desired state.
Manual production changes are exceptions to be reconciled, not permanent configuration.

State lives outside compute.
Servers and clusters may disappear.

Backups must cross failure domains.
A backup inside the system being backed up is insufficient.

Restore is the real backup test.
Successful backup jobs do not prove recoverability.

Applications use portable protocols.
Cloud-specific APIs stay at the infrastructure boundary.

Scaling must respect downstream capacity.
Ten more API pods should not accidentally create ten times as many PostgreSQL connections.

Health probes have distinct meanings.
Dependency failure should not create Kubernetes restart storms.

Failure tests are first-class artifacts.
Recovery assumptions should be machine-verifiable.

⸻

Roadmap

The architecture is complete at the specification level.

The next milestones are implementation:

* Terraform reference implementation for the Hetzner substrate;
* Talos machine configuration generation;
* Argo CD bootstrap;
* CloudNativePG + Barman recovery implementation;
* MinIO distributed deployment;
* immutable object snapshot pipeline;
* Tier-3 independent recovery;
* application runtime manifests;
* certification harness;
* bootstrap orchestrator;
* version-pinned sovereign-toolkit container;
* full staging destruction/rebuild test;
* provider-loss recovery test.

The reference implementation will evolve.

The five invariants should remain stable.

⸻

Contributing

Contributions are welcome, particularly where they improve:

recoverability
portability
failure testing
operational simplicity
security boundaries
provider independence
documentation

A proposed infrastructure change should answer:

1. What failure does this protect against?
2. What new dependency does it introduce?
3. How is recovery performed?
4. How is the claim tested?
5. Does it weaken any of the five sovereign invariants?

Tool substitutions are welcome when they improve the contract rather than merely increase the number of tools in the stack.

⸻

Project Philosophy

This project is deliberately skeptical of architecture that exists only on diagrams.

The central operating rule is:

Never trust architecture because the YAML looks correct. Trust only failure behavior you have deliberately induced, observed, measured, and successfully recovered from.

Git defines desired state.

Backups define durable state.

Independent recovery protects sovereignty.

Certification proves the difference.

⸻

Read the Full Playbook

→ Sovereign SaaS Playbook — Canonical Architecture

The playbook covers the complete system, including:

* physical and Kubernetes topology;
* Talos and Hetzner networking;
* GitOps;
* secrets management;
* PostgreSQL/CNPG;
* pgvector;
* PgBouncer;
* database PITR;
* Redis;
* Better Auth;
* tenant isolation and RLS;
* MinIO erasure coding;
* immutable object snapshots;
* Tier-2 and Tier-3 recovery;
* Next.js and Node.js runtime behavior;
* progressive delivery;
* autoscaling;
* observability;
* capacity testing;
* disaster recovery;
* certification protocols;
* upgrade policy;
* sovereignty testing.

⸻

License

A project license has not yet been specified.

Before accepting external contributions or encouraging production redistribution, add an explicit LICENSE file and update this section accordingly.