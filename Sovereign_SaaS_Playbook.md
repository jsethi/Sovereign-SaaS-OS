THE SOVEREIGN SAAS PLAYBOOK

Deploying a FOSS, AI-Ready SaaS Stack on Hetzner with Portable Infrastructure, Independent Recovery, and Machine-Verifiable Failure Contracts

Version: 3.0
Status: CANONICAL ARCHITECTURE
Primary implementation: Hetzner Cloud + Hetzner Dedicated
Execution substrate: Talos Linux + upstream Kubernetes
Application stack: Next.js + Node.js + Better Auth
Data stack: PostgreSQL + pgvector + Redis + MinIO
Control plane: Git + Argo CD
Observability: Prometheus + Grafana + Loki + Tempo/OpenTelemetry
Operating principle: Compute is disposable. State is recoverable. Recovery is independently verifiable.

⸻

0. SYSTEM PURPOSE

The Sovereign SaaS OS bridges the gap between open-source infrastructure and production-grade SaaS operations.

It is designed for teams that want the portability and control of open systems without giving up the operational discipline normally associated with managed cloud platforms.

The objective is not:

self-host everything.

The objective is:

build a SaaS platform whose compute can be destroyed, whose infrastructure provider can be replaced, whose durable state can be reconstructed, and whose recovery claims are continuously proven.

The system therefore treats Kubernetes, servers, pods, caches, and deployments as replaceable execution machinery.

The durable system consists of:

Git desired state
+
independent bootstrap secrets
+
PostgreSQL backups + WAL
+
immutable object recovery points
+
independent Tier-3 copies

Everything else must be reconstructible.

⸻

1. THE FIVE SOVEREIGN INVARIANTS

Every architectural decision must satisfy five invariants.

1.1 Rebuildability

Compute is ephemeral.

A destroyed cluster must be reconstructible from:

Terraform
+
Talos configuration
+
Git
+
bootstrap secrets
+
durable state

No production system may depend on undocumented manual configuration.

⸻

1.2 Recoverability

Every stateful system has:

* an explicit failure model;
* an explicit recovery mechanism;
* an RPO target;
* an RTO target;
* a destructive certification test.

A backup that has never been restored is not considered verified.

⸻

1.3 Portability

Application code does not know that Hetzner exists.

Applications depend only on portable interfaces such as:

HTTP
PostgreSQL wire protocol
S3-compatible object API
Redis protocol
Kubernetes workload primitives
environment-based configuration

Infrastructure portability does not mean Terraform is magically provider-neutral.

It means the infrastructure implementation can be replaced without redesigning the application domain model.

⸻

1.4 Operability

Operational procedures are encoded.

The repository contains runbooks for:

bootstrap
failover
restore
secret rotation
certificate rotation
provider migration
upgrade
incident response
capacity validation

No critical recovery procedure may depend exclusively on institutional memory.

⸻

1.5 Independence

No recovery mechanism may depend solely on the infrastructure, account, credentials, or failure domain that it exists to recover.

Examples:

Postgres running in Kubernetes
→ backup outside Kubernetes
MinIO running at Hetzner
→ recovery copy outside the MinIO cluster
Hetzner Tier-2
→ Tier-3 under a separate provider/account boundary

This is the defining difference between replication and sovereign recovery.

⸻

2. WHAT “SOVEREIGN” MEANS

Sovereignty does not mean having no vendors.

Every production system has dependencies.

The goal is instead:

maintain credible exit paths.

Hetzner is therefore an implementation provider rather than an application dependency.

The practical definition is:

A sovereign system can be rebuilt, recovered, verified, and relocated without requiring the continued cooperation of a single infrastructure provider.

⸻

3. HIGH-LEVEL ARCHITECTURE

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
              │                                           │
              ▼                                           ▼
     Hetzner Object Storage                    Immutable Snapshots
      Base backup + WAL                              Tier-2
              │                                           │
              └────────────────────┬──────────────────────┘
                                   ▼
                         Independent Provider
                          e.g. Backblaze B2
                               Tier-3
                         COMPLIANCE retention

⸻

4. PHYSICAL TOPOLOGY

The reference implementation uses a hybrid Hetzner topology.

Control Plane

3 × Hetzner Cloud nodes

Example class:

CX-class

These run:

Talos
Kubernetes control plane
etcd

No normal application workload is scheduled here.

⸻

Worker Plane

4 × Hetzner Dedicated AX-class servers

The workers provide:

CPU
RAM
local NVMe
stateful workload placement

Using four workers provides a natural failure topology for:

MinIO four-drive erasure set
CNPG anti-affinity
application workloads
background workers

Exact server models are implementation details and should be selected against the current Hetzner catalogue rather than permanently embedded in the architecture.

⸻

5. HETZNER NETWORK MODEL

The cluster uses one routed private underlay.

Hetzner Cloud Network
10.100.0.0/16
│
├── Cloud subnet
│   10.100.0.0/24
│
│   cp-1    10.100.0.11
│   cp-2    10.100.0.12
│   cp-3    10.100.0.13
│   API LB  10.100.0.10
│
└── vSwitch subnet
    10.100.1.0/24
    worker-1  10.100.1.21
    worker-2  10.100.1.22
    worker-3  10.100.1.23
    worker-4  10.100.1.24

Dedicated workers remain attached to their normal physical uplink.

Private traffic uses the Hetzner vSwitch VLAN on that uplink.

Do not model workers as if they have two independent private Kubernetes networks.

The pod network sits above the node network.

⸻

6. KUBERNETES NETWORKING

Recommended contract:

Pod CIDR:
10.244.0.0/16
Service CIDR:
10.96.0.0/12
CNI:
Cilium

Cilium provides:

pod networking
network policy
service dataplane
Gateway/API integration where configured
observability hooks

WireGuard/Tailscale-style admin networking is not used as the Kubernetes pod underlay.

Administrative networking is a separate concern.

⸻

7. TALOS LINUX

Talos runs upstream Kubernetes.

Do not combine Talos with K3s.

Talos provides the immutable node-management model; Kubernetes remains upstream.

The architectural contract is:

no conventional SSH administration
declarative machine configuration
Talos API administration
immutable host model
pinned Talos + Kubernetes versions

The repository pins specific release versions.

The playbook does not.

⸻

8. HETZNER CLOUD CONTROLLER

Talos enables Kubernetes external cloud-provider mode.

That does not itself install or configure Hetzner CCM.

HCCM is separately deployed and configured with the permissions required for the actual topology.

The hybrid topology may require both Cloud and dedicated/Robot integration according to the HCCM release used.

Credential scope must be minimized.

The Kubernetes workloads must never receive infrastructure-admin credentials.

⸻

9. NODE FAILURE DOMAINS

Worker nodes receive deterministic labels.

Example:

node.sovereign.io/role=stateful
node.sovereign.io/storage=nvme
node.sovereign.io/failure-domain=ax42-1

through:

ax42-4

These labels drive:

CNPG placement
MinIO placement
LocalPV placement
anti-affinity
failure tests

Scheduler defaults are not considered a durability contract.

⸻

10. LOCAL STORAGE

Each worker separates host OS storage from application state.

Conceptually:

NVMe 0
└── Talos
NVMe 1
└── Kubernetes workload storage
    ├── CNPG
    └── MinIO

The exact Talos volume API and LocalPV implementation are pinned by the implementation repository.

The invariant is:

every PVC must resolve to a known disk and known node failure domain.

A disk appearing in a Talos machine definition does not automatically make it Kubernetes storage.

⸻

11. GITOPS CONTROL PLANE

Git contains desired state.

Argo CD reconciles that state into Kubernetes.

Recommended structure:

infra/
  base/
  operators/
  database/
  minio/
  app/

Deployment order follows dependency boundaries rather than alphabetical ordering.

Example:

Wave 0
CRDs / operators
Wave 1
storage / certificates / networking dependencies
Wave 2
databases / object stores
Wave 3
runtime dependencies
Wave 4
application
Wave 5
autoscaling / analysis / certification

Production image references are immutable digests.

Never use:

:latest
:stable
:production

as desired state.

⸻

12. SECRET MANAGEMENT

Secrets committed to Git are encrypted with:

SOPS
+
age

The age private key is not committed.

Bootstrap retrieves it through an independent secrets path.

Secret classes include:

Talos bootstrap material
HCCM credentials
PostgreSQL credentials
MinIO credentials
Tier-2 backup credentials
Tier-3 writer credentials
Tier-3 restore credentials
Better Auth secrets
manifest-signing material

Secrets are separated according to operational responsibility.

A backup writer should not automatically be able to:

delete backups
change retention
manage buckets
administer Kubernetes
administer the provider account

⸻

13. POSTGRESQL

PostgreSQL is the authoritative relational datastore.

Deployment:

CloudNativePG
3 PostgreSQL instances

Typical topology:

primary
synchronous standby
additional replica

The contract requires at least one synchronous standby during normal operation.

CloudNativePG owns PostgreSQL’s synchronous replication configuration.

Do not manually fight the operator by hard-coding internal standby state the operator is intended to control.

⸻

14. PGVECTOR

pgvector runs inside PostgreSQL.

Do not globally prescribe:

IVFFlat probes = 10
HNSW ef_search = 40
HNSW always
IVFFlat above N rows

Index strategy is benchmark-driven.

Evaluate:

recall@k
p50/p95/p99 latency
index build duration
memory overhead
disk amplification
filter selectivity
vector dimensions
dataset size
write frequency

The chosen index configuration is committed as schema/migration state.

⸻

15. PGBOUNCER

Applications connect through the CNPG Pooler rather than directly to PostgreSQL.

Next.js / API / Workers
          │
          ▼
     CNPG Pooler
       PgBouncer
          │
          ▼
      PostgreSQL

For stateless HTTP workloads, transaction pooling should be the initial benchmark candidate unless the application depends on PostgreSQL session-level state.

CNPG provides a managed Pooler abstraction for PgBouncer rather than requiring applications to own the PgBouncer lifecycle directly. (CloudNativePG)

Do not assume an undocumented generated Service name.

Discover and validate it using CNPG labels.

⸻

16. DATABASE CONNECTION BUDGET

Autoscaling must never imply unlimited database connections.

Define:

max application replicas
×
max client pool per replica

and ensure the resulting system remains inside the PgBouncer client budget.

Then bound:

PgBouncer server connections
<
PostgreSQL max_connections
-
operational reserve

Reserve connections for:

CNPG
migrations
monitoring
manual incident response
maintenance

⸻

17. DATABASE BACKUP

CloudNativePG’s forward path uses the Barman Cloud Plugin for object-store backup/recovery. Current CNPG documentation recommends the plugin for object-store recovery, while older in-tree Barman configuration is being phased out. (CloudNativePG)

The architecture therefore uses:

CNPG Cluster
+
Barman Cloud Plugin
+
ObjectStore CRD
+
ScheduledBackup

The exact plugin CRD version is pinned.

Barman Cloud supports S3-compatible object stores, making an S3-compatible Hetzner target a valid implementation class. (CloudNativePG)

⸻

18. POSTGRESQL TIER-2

Database backups go directly outside Kubernetes.

PostgreSQL
   │
   ├── base backups
   └── WAL archive
             │
             ▼
    Hetzner Object Storage

Do not route the PostgreSQL recovery chain through the MinIO cluster it may need to survive.

Hetzner Object Storage and Storage Box are separate products; Object Storage is the S3-compatible primitive used here. (Hetzner Docs)

⸻

19. POSTGRESQL FAILURE CONTRACTS

Single Primary/Worker Failure

Target:

RPO:
0 acknowledged synchronous transactions
RTO:
< 60 seconds target

The exact RTO is measured.

⸻

Complete PostgreSQL Cluster Loss

Recovery:

new CNPG cluster
+
latest valid base backup
+
archived WAL replay

Target:

RPO:
bounded by durable WAL archive lag
RTO:
< 30 minutes target

Do not claim recovery to “the last committed transaction.”

The correct statement is:

recovery reaches the latest transaction represented in durable archived WAL or an explicitly selected PITR target.

⸻

Provider Failure

The independent database recovery chain must exist outside the Hetzner trust boundary.

Target RPO is determined by Tier-3 replication lag.

⸻

20. DATABASE RECOVERY CERTIFICATION

At minimum:

DB-001
Insert transaction T1
DB-002
Verify synchronous replica has T1
DB-003
Abruptly terminate primary
DB-004
Measure failover
DB-005
Verify T1 survives
DB-006
Insert T2
DB-007
Advance/archive WAL
DB-008
Destroy CNPG cluster
DB-009
Create recovery cluster
DB-010
Restore from Tier-2
DB-011
Validate PITR boundary
DB-012
Attempt protected-backup deletion
DB-013
Verify retention
DB-014
Restore independently from Tier-3

Environment RPO/RTO values come from these tests.

⸻

21. REDIS

Redis is disposable.

Deployment characteristics:

Deployment
no PVC
explicit maxmemory
explicit eviction policy

Redis may store:

rate-limit counters
reproducible short-lived state
non-authoritative caches

Redis is not permitted to become the only durable record for authentication or customer data unless the product explicitly accepts that failure contract.

⸻

22. BETTER AUTH

PostgreSQL remains authoritative for identities/accounts and, in this architecture, durable sessions.

Better Auth currently documents that when secondary storage is configured, sessions use secondary storage by default; session.storeSessionInDatabase: true keeps sessions in the database as well. (Better Auth)

Canonical intent:

betterAuth({
  database,
  secondaryStorage: redisSecondaryStorage,
  session: {
    storeSessionInDatabase: true,
    cookieCache: {
      enabled: true,
      maxAge: 300,
    },
  },
  secrets: [
    { version: 2, value: process.env.AUTH_SECRET_V2! },
    { version: 1, value: process.env.AUTH_SECRET_V1! },
  ],
})

The cookie cache is Better Auth’s cookie-side session caching mechanism.

It is not Redis.

Redis secondary storage is treated independently.

⸻

23. AUTH SECRET ROTATION

Use overlapping versioned secrets.

Sequence:

V1 active
introduce V2
V2 current
V1 retained
validate:
old auth material works
new auth material uses V2
wait compatibility window
remove V1

Do not rotate by simply replacing one secret value and hoping existing sessions survive.

⸻

24. TENANT ISOLATION

Authentication is not authorization.

The request path is:

JWT/session
     │
     ▼
authenticated user
     │
     ▼
tenant membership
     │
     ▼
application authorization
     │
     ▼
database tenant context
     │
     ▼
RLS

A client-controlled:

X-Tenant-ID

is never authoritative.

For PostgreSQL:

SET LOCAL app.tenant_id = 'tenant-123';

RLS is an independent safety boundary.

Certification must verify both:

application denial
database denial

⸻

25. MINIO LOCAL OBJECT STORAGE

MinIO provides the application’s S3-compatible object interface.

Topology:

4 servers
1 explicitly bound drive each
1 intended erasure set
EC:2

The manifest and certification suite must prove this actual topology rather than infer it from pod count.

⸻

26. MINIO FAILURE SEMANTICS

For a four-drive erasure set with two parity shards, read and write behavior differs once failures accumulate.

The canonical architecture therefore tests rather than merely documents quorum.

Expected high-level behavior:

1 node/drive lost
→ reads available
→ writes available
2 node/drives lost
→ reads may remain available
→ write quorum unavailable
3 lost
→ unavailable

Exact behavior is validated against the pinned MinIO release/configuration.

⸻

27. THREE DIFFERENT OBJECT PROTECTION PRIMITIVES

These concepts are deliberately separate.

Local availability

MinIO EC

protects local service against drive/node loss.

Local history

bucket versioning

protects previous versions while MinIO itself is intact.

External backup

immutable snapshots

provide independent recovery.

The rule is:

Replication copies state. Versioning preserves history within one storage system. Backup preserves independently recoverable states.

⸻

28. BUCKET BOOTSTRAP

Versioning is a bucket property.

Bucket initialization is handled by an idempotent bootstrap operation after MinIO becomes healthy.

It performs:

create bucket if missing
enable versioning
configure lifecycle
configure application quota
verify resulting state

Application quota and disk headroom are separate concepts.

Quota must not be described as direct physical disk reserve.

⸻

29. OBJECT SNAPSHOT MODEL

Every scheduled snapshot records a point:

T

The snapshotter determines the object namespace as of T.

It retrieves explicit historical VersionIds rather than reading whichever object happens to be current while the scan progresses.

A snapshot therefore represents:

the exact object versions selected at the snapshot boundary.

It does not automatically preserve every intermediate MinIO version created between two scheduled snapshots.

⸻

30. SNAPSHOT MANIFEST

A portable manifest stores application-significant object state.

Example:

{
  "schema": "sovereign-object-snapshot/v1",
  "snapshot_id": "2026-08-12T15:00:00Z",
  "source_bucket": "application-bucket",
  "objects": {
    "documents/foo.pdf": {
      "version_id": "source-version",
      "sha256": "abc123...",
      "size": 182731,
      "content_type": "application/pdf",
      "metadata": {}
    }
  }
}

Provider-native metadata is not automatically considered portable.

If an S3 property matters to the application, encode it in the manifest schema.

⸻

31. CONTENT ADDRESSING

Snapshot blobs are addressed by SHA-256:

objects/
  ab/
    cd/
      abcdef123...

Two keys containing identical bytes reference one content blob.

This provides deduplication at the logical backup layer.

It is independent of whether the S3 provider performs physical deduplication.

⸻

32. SNAPSHOT COMMIT PROTOCOL

A snapshot is not valid because a manifest file exists.

Publication is transactional at the protocol level.

1. record T
2. enumerate object state at T
3. retrieve explicit versions
4. compute hashes
5. upload missing content blobs
6. verify blobs
7. build manifest
8. sign manifest
9. upload manifest
10. upload detached signature
11. verify manifest references
12. write COMMIT object LAST

Only committed snapshots are eligible for restore.

⸻

33. SNAPSHOT INTEGRITY

For every committed recovery point:

manifest.json
manifest.json.sig
commit.json

Restore verifies:

manifest signature
manifest SHA-256
referenced blob SHA-256

The initial signing key may live in a SOPS-protected workload secret for corruption detection.

Higher maturity moves signing to:

KMS
HSM
offline signing
separate trust domain

so a compromised production cluster cannot fabricate valid backup history.

⸻

34. TIER-2 OBJECT RECOVERY

Tier-2 object snapshots live in Hetzner Object Storage.

Required properties:

S3-compatible access
versioning enabled
Object Lock
retention
external to Kubernetes
separate credentials

Hetzner provides object versioning/Object Lock functionality in Object Storage. (Hetzner Docs)

Storage structure:

objects/
manifests/
commits/

Backup cleanup is controlled by lifecycle policy.

The snapshot producer never runs a synchronization process that deletes older recovery points merely because production deleted an object.

⸻

35. TIER-3

Tier-3 exists under a separate administrative/provider boundary.

Reference implementation:

Backblaze B2
S3-compatible API
COMPLIANCE Object Lock

Backblaze’s S3-compatible API supports Object Lock, including compliance and governance retention modes. (Backblaze)

The transfer model is:

copy
not sync

because deletion propagation is undesirable in an immutable recovery tier.

⸻

36. TIER-3 CREDENTIALS

Separate principals:

Writer

read Tier-2
list Tier-3 if required
write Tier-3
no delete
no bucket administration
no retention weakening

Restorer

read Tier-3
no write
no delete

Administrator

bucket configuration
retention policy
kept outside ordinary workloads

Backblaze application keys support capability-based restriction, so the Tier-3 credentials should be scoped to the narrowest required operations. (Backblaze)

⸻

37. TIER-3 COPY MODEL

Tier-3 processing is driven by committed snapshots, not merely directories.

Algorithm:

discover Tier-2 commits
        │
        ▼
find commits without Tier-3 receipt
        │
        ▼
copy referenced blobs
        │
        ▼
copy manifest + signature
        │
        ▼
copy commit
        │
        ▼
verify destination
        │
        ▼
write replication receipt

The writer must never need retention-admin privileges.

⸻

38. TIER-3 RECEIPTS

Example:

{
  "snapshot_id": "2026-08-12T15:00:00Z",
  "manifest_sha256": "abc123...",
  "verified": true,
  "replicated_at": "2026-08-12T15:13:42Z"
}

Monitor:

latest committed Tier-2 snapshot
latest verified Tier-3 receipt
replication lag

The Tier-3 RPO is based on the newest verified receipt.

It is not based merely on the last CronJob execution.

⸻

39. MINIO CERTIFICATION

MINIO-001

Lose one node gracefully.

Verify:

GET succeeds
PUT succeeds
quorum healthy

MINIO-002

Lose one node abruptly.

Verify the same contract under ungraceful failure.

MINIO-003

Lose two nodes.

Verify documented read/write behavior.

MINIO-004

Restore nodes.

Verify healing and object hashes.

MINIO-005A

Mutate objects during snapshot generation.

Restore snapshot.

Verify no selected object version exceeds snapshot boundary.

MINIO-005B

Kill snapshot midway.

Verify:

no valid commit
snapshot excluded from recovery list

MINIO-005C

Restore historical committed snapshots.

Verify object namespace and content at each recovery point.

MINIO-005D

Store identical data under two keys.

Verify:

two manifest entries
one content-addressed blob

MINIO-006

Remove access to Hetzner Tier-2.

Restore from Tier-3 with no Hetzner backup credentials.

Verify complete reconstruction.

⸻

40. NEXT.JS

Next.js is stateless.

Build production images with standalone output where appropriate.

Configuration comes from runtime environment.

No secret is embedded in the image.

Production images use immutable digests.

Scaling metrics should reflect actual server workload, such as:

requests/sec
in-flight SSR requests
CPU
event-loop pressure

⸻

41. NODE API

Node.js API services are stateless.

Responsibilities include:

authentication orchestration
authorization
tenant context
application APIs
RAG/vector orchestration
background job submission
S3 interactions

The API never imports Hetzner-specific application SDKs.

⸻

42. APPLICATION HEALTH CONTRACT

Every runtime exposes three probes.

Startup

/health/startup

Means:

initialization has completed.

Liveness

/health/live

Means:

process/event loop is functioning.

A database outage does not automatically fail liveness.

Readiness

/health/ready

Means:

this instance is currently safe to receive traffic.

Database/pool backpressure may cause readiness to fail after a bounded threshold.

This prevents dependency outages from creating pod restart storms.

⸻

43. GRACEFUL SHUTDOWN

Typical contract:

terminationGracePeriodSeconds: 35

On SIGTERM:

mark readiness false
stop accepting new work
stop queue acquisition
drain bounded in-flight requests
close application pools
exit before grace deadline

A short PreStop delay may be used for endpoint propagation.

It is not the primary shutdown implementation.

⸻

44. PROGRESSIVE DELIVERY

Use Argo Rollouts.

Without an integrated traffic router, canary weight is a replica approximation.

With three replicas:

1 canary
≈ 33%
2 canary
≈ 66%
3 canary
= 100%

Do not claim precise request percentages unless traffic routing actually enforces them.

Canary analysis should include:

5xx ratio
p95 latency
readiness failures
PgBouncer saturation
event-loop lag

Failure automatically aborts the rollout.

⸻

45. CANARY SLO ANALYSIS

Prometheus returns the actual metric.

Example conceptual error ratio:

5xx request rate
/
all request rate

The rollout evaluator decides whether that ratio violates the SLO.

Do not use ambiguous boolean PromQL where numeric results provide better diagnostics.

⸻

46. AUTOSCALING

HPA signals:

requests/sec
in-flight requests
CPU

Counters such as:

http_requests_total

must be converted to rates.

Prometheus metrics do not automatically become Kubernetes custom metrics.

The cluster therefore explicitly includes a custom-metrics adapter.

Certification includes:

kubectl get --raw /apis/custom.metrics.k8s.io/

before enabling HPA dependent on custom metrics.

⸻

47. RATE LIMITING

Use several buckets.

IP
→ unauthenticated abuse
user_id
→ user fairness
tenant_id
→ commercial plan limits
API key
→ machine clients
route + tenant
→ expensive operations

Example:

tenant:abc:embedding
max concurrent:
20
hourly:
500

Redis can implement these counters because they are disposable/reconstructible.

⸻

48. DATABASE BACKPRESSURE

PgBouncer exhaustion must not become retry amplification.

When database capacity is unavailable:

fail quickly
return controlled 503/429 as appropriate
use bounded retries
apply jitter
protect Postgres from reconnection storms

Circuit breakers may be used around:

embedding services
external APIs
long-running vector operations

They are not substitutes for capacity engineering.

⸻

49. NETWORK POLICY

Application workloads use least-privilege network access.

Ingress examples:

Next.js → API
Gateway → appropriate frontend/API

Egress examples:

DNS
CNPG Pooler
Redis
MinIO
required third-party APIs

Default-deny egress also affects DNS unless DNS is explicitly allowed. Kubernetes NetworkPolicies are additive, so adding a second policy does not revoke traffic allowed by another policy. Fault injection must account for those semantics.

⸻

50. OBSERVABILITY

Use the RED method:

Rate
Errors
Duration

Also monitor:

tenant-scoped failures
PgBouncer saturation
PostgreSQL replication lag
WAL archive lag
MinIO quorum
disk utilization
snapshot duration
snapshot age
Tier-3 receipt lag
Redis availability
event-loop lag
rollout state

⸻

51. LOGGING

Applications emit structured JSON.

Required contextual fields:

request_id
trace_id
user_id where appropriate
tenant_id
route
status
duration
deployment version

Do not log:

secrets
authorization headers
raw session tokens
sensitive user payloads

Loki provides central log aggregation.

⸻

52. TRACING

OpenTelemetry instruments:

Next.js server execution
Node API requests
database queries
vector search
S3 calls
external API calls
background jobs

Tempo or another OpenTelemetry-compatible backend stores traces.

RAG performance should be visible as an end-to-end trace rather than isolated service timings.

⸻

53. ALERTING

P0 examples:

database unavailable
MinIO write quorum lost
Tier-2 recovery unavailable

P1 examples:

WAL archive stale
Tier-3 receipt lag excessive
storage above critical threshold
persistent rollout aborts

Alerting must reflect actionable failure contracts.

⸻

54. CAPACITY GOVERNANCE

Never say:

50k users = X servers

MAU does not determine capacity.

Benchmark:

HTTP

RPS
concurrency
p50
p95
p99
5xx rate

PostgreSQL

TPS
working set
PgBouncer clients
PgBouncer servers
replication lag
CPU
IOPS

Vector

vector count
dimensions
top-k
filter selectivity
recall@k
p95 latency
index size

Objects

GET/s
PUT/s
MB/s
object count
disk utilization
healing time

Recovery

DB GB/min
WAL replay rate
object restore GB/min
full-system RTO

Capacity claims are valid only against the benchmarked workload/version set.

⸻

55. COST GOVERNANCE

Hetzner is selected because dedicated compute and network economics can be attractive.

But the OS deliberately avoids fixed claims such as:

$500 for 50,000 users

Actual cost depends on:

traffic
concurrency
storage
IO
database workload
vector workload
backup size
egress
availability target

Cost optimization follows measurement rather than marketing estimates.

⸻

56. BOOTSTRAP ORDER

A new environment is constructed in this order.

1. provision Tier-3 destination
2. provision Tier-2 recovery stores
3. provision Hetzner network
4. provision control-plane compute
5. provision dedicated workers
6. configure vSwitch
7. generate/apply Talos config
8. bootstrap Kubernetes
9. install Cilium
10. install HCCM
11. install cert-manager
12. bootstrap Argo CD
13. install operators/CRDs
14. configure LocalPV
15. deploy CNPG
16. configure Barman Cloud backups
17. deploy PgBouncer Pooler
18. deploy MinIO
19. bootstrap buckets
20. enable immutable snapshot pipeline
21. enable Tier-3 copy/verification
22. deploy Redis
23. deploy Node API
24. deploy Next.js
25. configure monitoring
26. configure Prometheus Adapter
27. enable HPA
28. run certification suites
29. enable production traffic

Infrastructure existing is not the same as infrastructure being ready.

⸻

57. COMPLETE CLUSTER LOSS RUNBOOK

Assume:

Kubernetes gone
workers gone
Tier-2 available
Git available
bootstrap secrets available

Procedure:

1. provision replacement infrastructure
2. bootstrap Talos/Kubernetes
3. install Cilium + HCCM
4. bootstrap Argo
5. reconcile operators
6. create replacement CNPG cluster
7. restore DB from base backup + WAL
8. deploy empty MinIO
9. identify latest valid Tier-2 commit
10. verify manifest/signature
11. restore objects through S3 API
12. deploy empty Redis
13. deploy application
14. run synthetic tests
15. run tenant isolation checks
16. open traffic

MinIO internal filesystem contents are never manually reconstructed.

⸻

58. PROVIDER LOSS RUNBOOK

Assume:

Hetzner gone
Hetzner APIs inaccessible
Hetzner Object Storage unavailable

The recovery test must not use:

HCLOUD_TOKEN
Robot credentials
Hetzner S3 credentials

Procedure:

1. provision Kubernetes elsewhere
2. deploy equivalent networking/storage primitives
3. restore PostgreSQL from independent copy
4. deploy empty S3-compatible object store
5. identify latest Tier-3 verified receipt
6. retrieve commit
7. retrieve manifest/signature
8. verify cryptographic integrity
9. restore content-addressed objects
10. reconstruct object namespace
11. deploy Redis
12. deploy applications
13. run certification
14. expose traffic

Success means:

the application serves validated traffic without Hetzner.

⸻

59. UPGRADE POLICY

Pin:

Talos
Kubernetes
Cilium
HCCM
cert-manager
Argo CD
Argo Rollouts
CNPG
Barman Cloud Plugin
MinIO Operator
MinIO
Prometheus stack
Prometheus Adapter
Redis
rclone
mc
application images

Upgrade workflow:

read release notes
        ↓
update staging
        ↓
run certification
        ↓
perform recovery drill where relevant
        ↓
deploy canary
        ↓
observe
        ↓
promote
        ↓
record version set

CRDs and controllers are upgraded together.

⸻

60. CERTIFICATION TERMINOLOGY

Use these terms precisely.

Canonical

The architecture/specification has been approved.

Certification-ready

Executable tests exist.

Certified

Required tests passed against the actual deployed version/environment.

Continuously certified

Tests are repeated on schedule and results retained.

Do not label a platform certified merely because test scripts exist.

⸻

61. SUBSTRATE CERTIFICATION

Test:

control-plane quorum
worker registration
Cloud↔vSwitch routing
Cilium pod connectivity
DNS
HCCM behavior
LocalPV node binding
failure-domain scheduling
API endpoint failover

⸻

62. DATABASE CERTIFICATION

Test:

sync transaction survives primary loss
failover completes
WAL archive advances
PITR works
full cluster restore works
protected objects cannot be removed early
Tier-3 restore works
RPO measured
RTO measured

⸻

63. OBJECT CERTIFICATION

Test:

one-node graceful loss
one-node abrupt loss
two-node failure semantics
healing
hash integrity
disk pressure alerting

⸻

64. SNAPSHOT CERTIFICATION

Test:

snapshot boundary consistency
interrupted snapshot invalid
PITR snapshot restore
namespace deletions respected
content deduplication
signature verification
blob verification

⸻

65. TIER-3 CERTIFICATION

Test:

T3-001
committed snapshot copied
T3-002
existing immutable object not rewritten
T3-003
destination mismatch fails loudly
T3-004
writer cannot delete protected object
T3-005
writer cannot reduce retention
T3-006
restore succeeds without Hetzner
T3-007
excess receipt lag generates alert

⸻

66. APPLICATION CERTIFICATION

APP-001

Abruptly kill exactly one Next.js pod.

Synthetic traffic must remain within SLO.

⸻

APP-002

Gracefully terminate exactly one API pod under active requests.

Verify:

readiness drops
new traffic stops
in-flight requests finish
process exits within grace period

⸻

APP-003

Deploy an intentionally bad canary.

Verify:

analysis fails
rollout aborts
stable revision remains serving

⸻

APP-004

Constrain PgBouncer/database capacity.

Verify:

controlled load shedding
bounded retries
no connection storm
Postgres remains protected

⸻

APP-005

Remove Redis.

Verify actual documented Better Auth behavior for the enabled features.

With DB-backed sessions configured, authoritative session state remains durable in PostgreSQL. Better Auth documents the DB-storage override via storeSessionInDatabase. (Better Auth)

⸻

APP-006

Rotate Better Auth secrets using overlapping versions.

Verify:

old material accepted during transition
new material uses current key
old key removed only after expiry window

⸻

APP-007

Scale API from minimum to maximum intended replicas.

Verify PostgreSQL backend connections remain inside the connection budget.

⸻

APP-008

Terminate a pod under active traffic.

Record:

termination timestamp
readiness-off timestamp
last newly accepted request
last completed request
process exit

⸻

APP-009

Make the database endpoint unreachable using a controlled fault mechanism.

Verify:

Ready=False
restartCount unchanged
/health/live remains healthy

Do not infer liveness from a nonexistent Pod Live condition.

⸻

APP-010

Tenant A attempts to access Tenant B.

Verify:

application authorization rejects
AND
RLS independently rejects

⸻

67. CHAOS POLICY

Chaos testing is not random destruction.

Each test corresponds to a declared failure contract.

Examples:

kill primary
lose worker
lose two MinIO workers
block DB access
kill snapshot job
invalidate Tier-2 access
remove Redis
deploy bad application revision

Each experiment must state:

expected failure
expected user impact
expected recovery
expected alert
maximum permitted RPO
maximum permitted RTO

⸻

68. RECOVERY SCHEDULING

Recommended cadence:

application failure tests
continuous / deployment-time
database failover
monthly
object failure/heal test
monthly
snapshot restore
monthly
Tier-3 restore
quarterly
full cluster rebuild
quarterly
provider-loss simulation
at least annually
or after major platform changes

Cadence should reflect risk and business requirements.

⸻

69. RUNBOOKS AS CODE

Required operational documents:

docs/
  BOOTSTRAP.md
  DATABASE_FAILOVER.md
  DATABASE_PITR.md
  OBJECT_RECOVERY.md
  CLUSTER_LOSS.md
  PROVIDER_LOSS.md
  SECRET_ROTATION.md
  CERT_ROTATION.md
  UPGRADE_POLICY.md
  INCIDENT_RESPONSE.md
  CAPACITY_TESTING.md

A runbook should include:

trigger
preconditions
commands
assertions
rollback
success condition
failure escalation

⸻

70. RECOMMENDED REPOSITORY STRUCTURE

.
├── README.md
├── Makefile
├── Taskfile.yml
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
│   ├── certificates/
│   ├── database/
│   ├── minio/
│   ├── redis/
│   ├── monitoring/
│   └── app/
│
├── apps/
│   ├── web/
│   ├── api/
│   └── auth/
│
├── scripts/
│   ├── bootstrap.sh
│   ├── restore-postgres.sh
│   ├── restore-object-snapshot.sh
│   ├── verify-snapshot.sh
│   ├── tier3-copy.sh
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

⸻

71. BOOTSTRAP ORCHESTRATOR

The final executable form should expose a small number of high-level commands.

Example:

make bootstrap
make certify
make certify-database
make certify-storage
make certify-app
make restore-tier2
make restore-tier3
make destroy-staging
make rebuild-staging

Every stage has assertions.

Example:

terraform applied
        ↓
network verified
        ↓
Talos healthy
        ↓
Kubernetes healthy
        ↓
operators ready
        ↓
database healthy
        ↓
backup verified
        ↓
MinIO healthy
        ↓
snapshot committed
        ↓
Tier-3 receipt exists
        ↓
application healthy
        ↓
certification passed

Failure stops the pipeline.

⸻

72. TOOLKIT IMAGE

To eliminate local dependency variance, build a version-pinned utility image containing:

talosctl
kubectl
helm
argocd
mc
rclone
psql
openssl
jq
curl
terraform/tofu if desired
certification scripts

Example:

ghcr.io/yourorg/sovereign-toolkit@sha256:...

Use the same toolkit in:

CI
operator laptops
disaster-recovery environments
certification jobs

⸻

73. PRODUCTION READINESS GATE

Production traffic is allowed only when:

Git clean
immutable image digests
secrets encrypted
network policy active
database healthy
synchronous replication healthy
WAL archive current
Tier-2 backup current
object snapshot current
Tier-3 receipt current
critical certification suites passing
no unresolved P0/P1
runbooks current

Recovery readiness is a production feature.

A system serving requests while backups are broken is degraded even if users have not yet noticed.

⸻

74. THE SOVEREIGNTY TEST

The ultimate certification is reconstruction.

Test A — Cluster Sovereignty

destroy staging
        ↓
terraform apply
        ↓
Talos bootstrap
        ↓
Argo bootstrap
        ↓
restore state
        ↓
deploy applications
        ↓
run certification
        ↓
serve synthetic traffic

Test B — Provider Sovereignty

remove Hetzner recovery credentials
        ↓
provision elsewhere
        ↓
restore database from independent copy
        ↓
restore objects from Tier-3
        ↓
deploy application
        ↓
run certification
        ↓
serve traffic

If these tests pass, sovereignty is demonstrated rather than asserted.

⸻

75. WHAT THIS OS DELIBERATELY DOES NOT CLAIM

It does not claim:

zero vendors
zero operational work
automatic infinite scaling
universal $/user economics
zero data loss under every imaginable event
zero downtime under every failure
perfect infrastructure portability

It claims something more defensible:

failure behavior is defined, recovery paths are independent, application interfaces are portable, infrastructure is replaceable, and those claims can be tested.

⸻

76. FINAL OPERATING MANDATE

The Sovereign SaaS OS follows one rule above all others:

Never trust architecture because the YAML looks correct. Trust only failure behavior you have deliberately induced, observed, measured, and successfully recovered from.

Git defines desired state.

PostgreSQL and object stores contain durable state.

Independent copies protect against provider failure.

Runbooks define recovery.

Certification proves the contracts.

The Kubernetes cluster is only the executor.

⸻

77. CANONICAL STATUS

The architecture is now frozen at the contract level.

Future changes should fall into one of three categories:

IMPLEMENTATION CHANGE
new component version or CRD syntax
does not alter invariant
PERFORMANCE CHANGE
new sizing / benchmark result
does not alter failure model
ARCHITECTURAL CHANGE
changes dependency boundary,
durability model,
recovery model,
or sovereign invariant

Only the third category requires a new architectural version.

Everything else belongs in the implementation repository.

⸻

78. NEXT DELIVERABLES

The architecture is complete.

The next work is execution:

1. Terraform reference implementation
2. Talos machine-config generation
3. Argo bootstrap
4. Make/Task bootstrap orchestrator
5. certification harness
6. sovereign-toolkit image
7. first full staging rebuild
8. first Tier-3 provider-loss restore

Once the first complete rebuild and independent restore pass, the environment may legitimately be labeled:

Sovereign SaaS OS — Certified