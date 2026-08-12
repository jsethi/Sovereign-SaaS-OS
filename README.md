<div align="center">

🛡️ Sovereign SaaS OS

A production-grade, open-source architecture for building SaaS without surrendering your infrastructure.

Portable infrastructure · Independent recovery · Machine-verifiable failure contracts

<br />

Compute is disposable. State is recoverable. Recovery is independently verifiable.

<br />

📖 Read the Playbook · 🧭 Architecture · 🚀 How to Use It · 🧪 Certification

</div>

⸻

Why this exists

Modern SaaS infrastructure tends to drift toward one of two extremes.

On one side, you get a pile of self-hosted open-source tools that technically works — until a node dies, a backup is corrupt, a secret leaks, or the one person who understands the cluster disappears.

On the other, you get a deeply integrated cloud architecture built from proprietary managed services that is operationally convenient but expensive, difficult to leave, and increasingly coupled to one provider.

Sovereign SaaS OS is an attempt at a third path.

It combines open infrastructure with the operational discipline expected from serious production systems:

* deterministic rebuilds;
* explicit RPO and RTO targets;
* independent backups;
* GitOps;
* immutable infrastructure;
* least-privilege credentials;
* provider-independent recovery;
* destructive failure testing;
* portable application contracts.

The goal is not to self-host everything.

The goal is to maintain credible exit paths.

⸻

What does “sovereign” mean?

Sovereignty does not mean having no vendors.

It means vendors remain replaceable implementations, rather than becoming part of your application architecture.

A sovereign system should be able to:

rebuild, recover, verify, and relocate without requiring the continued cooperation of a single infrastructure provider.

The application relies on portable interfaces:

HTTP
PostgreSQL
S3-compatible object storage
Redis
Kubernetes
Environment-based configuration

Hetzner is the reference infrastructure provider.

It is not supposed to become part of the product’s domain model.

⸻

🧭 The Five Invariants

Every architectural decision in Sovereign SaaS OS is evaluated against five rules.

Invariant	Meaning
🔁 Rebuildability	Compute can disappear. Git + secrets + durable state can recreate it.
🧯 Recoverability	Every stateful system has an explicit RPO, RTO, recovery mechanism, and restore test.
📦 Portability	Application code does not depend on Hetzner-specific APIs or proprietary infrastructure.
🛠️ Operability	Bootstrap, recovery, upgrades, and rotation are encoded instead of living in someone’s head.
🌍 Independence	Recovery must survive failure of the provider, account, or system being recovered.

A backup that has never been restored is not considered verified.

⸻

🏗️ Architecture

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
                     └──────────────┬────────────┘
                                    │
                         Auth / Authorization
                           Tenant Context
                             Rate Limits
                                    │
              ┌─────────────────────┼─────────────────────┐
              │                     │                     │
              ▼                     ▼                     ▼
          PgBouncer              Redis                  MinIO
          CNPG Pooler          disposable             distributed
              │                                         EC:2
              ▼                                           │
         PostgreSQL                                       │
        3 instances                                       │
              │                                           ▼
              │                                 Immutable Snapshots
              ▼                                         Tier 2
     Hetzner Object Storage                               │
       Base Backup + WAL                                  │
              │                                           │
              └────────────────────┬──────────────────────┘
                                   ▼
                          Independent Provider
                           e.g. Backblaze B2
                                Tier 3
                         COMPLIANCE retention

Reference stack

Layer	Technology
Host OS	Talos Linux
Kubernetes	Upstream Kubernetes
Networking	Cilium
GitOps	Argo CD
Database	PostgreSQL + CloudNativePG
Vector search	pgvector
Pooling	PgBouncer
Cache	Redis
Object storage	MinIO
Authentication	Better Auth
Frontend	Next.js
API	Node.js
Metrics	Prometheus
Dashboards	Grafana
Logs	Loki
Tracing	OpenTelemetry / Tempo
Tier-2 backup	Hetzner Object Storage
Tier-3 recovery	Independent S3 provider

⸻

🔐 Recovery is part of the architecture

Most infrastructure diagrams stop at production state.

This project does not.

Application
    │
    ▼
Primary State
    │
    ▼
External Recovery State
    │
    ▼
Independent Recovery State
    │
    ▼
Verified Restore

PostgreSQL

PostgreSQL
    │
    ├── Base backups
    └── WAL archive
            │
            ▼
   Hetzner Object Storage
            │
            ▼
     Independent Tier 3

Object storage

MinIO
  │
  ├── Erasure coding
  ├── Local version history
  │
  ▼
Immutable point-in-time snapshots
  │
  ▼
Hetzner Object Storage
  │
  ▼
Independent compliance-locked Tier 3

Replication copies state. Versioning preserves history inside a storage system. Backup preserves independently recoverable states.

⸻

🧪 Failure contracts, not hope

The project is designed around deliberate destructive testing.

Instead of asking:

“Does the architecture look highly available?”

we ask:

“What happens when we actually break it?”

Examples:

Kill the PostgreSQL primary
→ Did the acknowledged transaction survive?
Destroy the PostgreSQL cluster
→ Can PITR rebuild it from external WAL?
Lose one MinIO node
→ Do reads and writes continue?
Lose two MinIO nodes
→ Does the expected quorum behavior occur?
Kill a snapshot job halfway through
→ Is the incomplete recovery point rejected?
Remove all Hetzner recovery credentials
→ Can Tier 3 still reconstruct the system?
Kill an API pod during active requests
→ Do requests drain correctly?
Make PostgreSQL unreachable
→ Does readiness fail without a restart storm?
Attempt cross-tenant access
→ Do both application authorization and RLS reject it?

The end goal is not merely Infrastructure as Code.

It is:

Failure behavior as code.

⸻

👥 Who this is for

This project is aimed at:

Technical founders

Building serious SaaS products who want more control than a PaaS without immediately committing to a hyperscaler-heavy architecture.

Small platform teams

Teams comfortable operating Kubernetes, PostgreSQL, and object storage who want a disciplined production baseline.

AI SaaS builders

Especially systems using:

PostgreSQL
pgvector
RAG
object storage
background jobs
APIs
AI pipelines

Organizations concerned about cloud concentration

Teams that want an actual provider-exit strategy rather than a diagram labelled “multi-cloud.”

Engineers who want infrastructure ownership

Without pretending that self-hosting magically removes operational complexity.

⸻

Who this is probably not for

Sovereign SaaS OS may be unnecessary if:

* your application is comfortably served by Vercel, Render, Railway, Fly.io, Supabase, or another PaaS;
* you don’t need Kubernetes;
* your team doesn’t want to operate stateful infrastructure;
* minimizing operational responsibility matters more than portability;
* your current scale does not justify this complexity.

Sovereignty has a cost.

This project is about making that cost explicit, structured, and testable.

⸻

🇩🇪 Why Hetzner?

Hetzner is the reference implementation because it offers a strong combination of:

* inexpensive cloud VMs;
* high-performance dedicated servers;
* private networking;
* load balancers;
* S3-compatible object storage;
* excellent price/performance.

But the architecture deliberately limits how far Hetzner reaches into the application.

The portability goal is roughly:

Layer	Portability
Application code	🟢 Very high
Kubernetes workloads	🟢 High
Platform services	🟡 Moderate–high
Terraform / infrastructure implementation	🟠 Provider-specific

Moving clouds may require significant infrastructure work.

It should not require rewriting the product.

⸻

🚀 How to use this project

1. Read the canonical playbook

Start here:

→ Sovereign SaaS Playbook

It contains the complete architecture, reasoning, failure models, recovery contracts, and certification philosophy.

⸻

2. Understand the invariants before copying YAML

This is not intended to be a bag of Helm charts.

The components are replaceable.

The contracts are the important part.

You can substitute:

Hetzner       → another provider
MinIO         → another S3-compatible store
Better Auth   → another auth system
Next.js       → another frontend
Node.js       → another API runtime

The five invariants should remain intact.

⸻

3. Build the reference implementation

The planned bootstrap flow is:

Terraform
    │
    ▼
Talos
    │
    ▼
Kubernetes
    │
    ▼
Cilium + HCCM
    │
    ▼
Argo CD
    │
    ▼
Operators
    │
    ▼
PostgreSQL
    │
    ▼
MinIO
    │
    ▼
Recovery pipelines
    │
    ▼
Applications
    │
    ▼
Certification

The eventual developer experience is intended to look roughly like:

make bootstrap
make certify
make certify-database
make certify-storage
make certify-app
make restore-tier2
make restore-tier3
make destroy-staging
make rebuild-staging

⸻

📁 Repository structure

Today, this repository contains the canonical specification.

The reference implementation will evolve toward:

.
├── README.md
├── Sovereign_SaaS_Playbook.md
├── LICENSE
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

✅ Canonical vs. Certified

These terms have specific meanings in this project.

Status	Meaning
Canonical	The architecture or contract has been approved.
Certification-ready	Executable tests exist.
Certified	Those tests passed against an actual deployed environment.
Continuously certified	Certification is repeated on schedule and results are retained.

A YAML file existing in Git does not make infrastructure certified.

⸻

🩺 Production readiness

A system isn’t healthy simply because it returns HTTP 200.

Production readiness includes:

Application healthy
Database healthy
Synchronous replication healthy
WAL archive current
Object snapshot current
Tier-3 recovery copy current
Secrets manageable
Network policy validated
Restore procedures tested
Failure contracts passing

If production is serving traffic but its recovery chain is broken, the system is degraded.

⸻

🧠 Design principles

Git defines desired state

Manual production changes are exceptions, not permanent configuration.

Compute is replaceable

Servers and Kubernetes clusters are not authoritative state.

Backups cross failure domains

A backup inside the thing being backed up is not enough.

Restore is the real backup test

A successful backup job proves that a backup job ran.

It does not prove recovery.

Application interfaces stay portable

Provider-specific APIs stop at the infrastructure boundary.

Scaling respects downstream systems

Ten more API pods should not accidentally create ten times more database connections.

Health checks have different meanings

Readiness, liveness, and startup are separate contracts.

Failure tests are first-class artifacts

Recovery assumptions should be executable.

⸻

🗺️ Roadmap

The architectural specification is complete.

The next milestones are implementation:

* Terraform reference implementation
* Hetzner Cloud + vSwitch networking
* Talos machine configuration generation
* Cilium + HCCM bootstrap
* Argo CD GitOps bootstrap
* CloudNativePG + Barman recovery
* MinIO distributed storage
* Immutable object snapshot pipeline
* Tier-3 provider-independent recovery
* Application runtime manifests
* Certification harness
* Bootstrap orchestrator
* Version-pinned sovereign-toolkit image
* Full staging destruction / rebuild
* Complete provider-loss recovery drill

The implementation will evolve.

The invariants should remain stable.

⸻

🤝 Contributing

Contributions are welcome.

Especially useful contributions improve:

* recoverability;
* portability;
* failure testing;
* security boundaries;
* operational simplicity;
* documentation;
* provider independence.

Before proposing a new component, ask:

1. What failure does this solve?
2. What dependency does it introduce?
3. How is the system recovered when it fails?
4. How is that claim tested?
5. Does it weaken any sovereign invariant?

This project deliberately avoids tool accumulation for its own sake.

⸻

📖 Read the full playbook

The complete specification is here:

→ The Sovereign SaaS Playbook

It covers:

* Hetzner Cloud + dedicated topology;
* Talos Linux;
* Kubernetes;
* Cilium networking;
* GitOps;
* secrets management;
* PostgreSQL and CloudNativePG;
* pgvector;
* PgBouncer;
* PITR and WAL recovery;
* Redis;
* Better Auth;
* multi-tenant RLS;
* MinIO erasure coding;
* immutable snapshots;
* Tier-2 Object Lock;
* Tier-3 independent recovery;
* Next.js and Node.js runtime behavior;
* progressive delivery;
* autoscaling;
* observability;
* chaos testing;
* RPO/RTO contracts;
* disaster recovery;
* certification.

⸻

📜 License

Sovereign SaaS OS is released under the MIT License.

See LICENSE.

⸻

<div align="center">

Sovereignty is not the absence of dependencies.

It is the maintained ability to leave them.

<br />

Never trust architecture because the YAML looks correct.
Trust the failures you have deliberately induced, measured, and successfully recovered from.

</div>