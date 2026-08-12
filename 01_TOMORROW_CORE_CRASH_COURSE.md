# Tomorrow Core Crash Course — MSD Sr. Enterprise Storage & DevOps

## How to use this tonight

Read in order. For each section: read once, close the file, draw the data path,
say the 30-second answer aloud, then answer the senior follow-up. Do not memorize
fake stories.

Evidence labels:

- **LAB-PRACTICED:** personally configured/tested in a lab.
- **TRAINED:** learned in structured training, without retained practice evidence.
- **CONCEPTUAL:** understand the design but have not configured it.
- **VERIFIED EXPERIENCE:** use only for facts you can support.

When personal production ownership is unverified, say so briefly, then show your
engineering method. A lab proves lab competence—not scale, customer impact,
on-call work, or a historical incident.

## The master storage picture

```text
Application
    |
    +-- file I/O --> filesystem/client --> NFS or SMB --> file service
    |
    +-- block I/O -> filesystem -> block device -> iSCSI or FC -> LUN
    |
    +-- object API -----------------------> HTTPS/API --> object store
                                                   |
                                                   v
                    controller/cache -> pool/aggregate -> RAID/media
```

Troubleshoot the path; never begin with “storage is slow.”

---

## 1. Enterprise storage fundamentals

### Simple explanation and memory hook

Storage is a service that preserves data and exposes it through an access model.
The three fundamental models are:

- **BLOCK = raw addressable chunks**
- **FILE = paths and folders**
- **OBJECT = key + data + metadata**

| Dimension | Block | File | Object |
|---|---|---|---|
| Client sees | Block device | Shared namespace | Bucket/container and object key |
| Access | SCSI-like reads/writes | NFS or SMB file operations | API, commonly HTTPS |
| Filesystem owner | Client/host | Storage/file server | Not exposed as a normal filesystem |
| Typical sharing | One host/cluster coordination; exceptions exist | Many clients | Many API clients |
| Metadata | Limited at block layer | Filesystem metadata | Rich per-object metadata |
| Common workloads | Databases, VM disks, boot/data volumes | Home dirs, shared apps, content | Backup, archive, data lake, media |
| AWS mapping | EBS | EFS; FSx families | S3 |

### How it works

Block storage returns numbered blocks. The host creates a filesystem or database
layout. File storage owns the filesystem and resolves names, directories,
permissions and locks for clients. Object storage accepts whole-object API
operations; clients address an object by key rather than mounting a native disk
filesystem.

### Production view

Storage selection begins with workload requirements—not vendor preference:

1. access semantics and concurrent writers;
2. latency, IOPS, throughput and I/O size;
3. capacity and growth;
4. availability, durability, backup, RPO and RTO;
5. security, identity, encryption and audit;
6. operations, lifecycle, portability and cost.

“Database on S3” needs precision. A conventional database normally expects
low-latency random block/file operations, locking and update-in-place semantics;
it cannot usually place its live data files directly on S3. A database engine
can deliberately use S3 for backups, exports, immutable objects or an
object-aware architecture.

### 30-second interview answer

> Block exposes raw addressable storage and leaves filesystem responsibility to
> the host. File exposes a shared filesystem through protocols such as NFS or
> SMB. Object exposes objects through an API using a key and rich metadata. I
> choose among them based on access semantics first, then sharing, performance,
> protection, security, operations and cost. EBS, EFS and S3 are AWS examples,
> but they are not interchangeable.

### Senior follow-up

**Why EBS instead of EFS?** Choose EBS for a host-attached block workload that
needs block semantics and predictable volume performance. Choose EFS for shared
managed NFS access from multiple clients. Validate AZ/topology, access mode,
throughput and recovery—not just headline latency.

### Troubleshooting angle

Identify the model before choosing tools. `lsblk` and filesystem tools help on
block; `nfsstat`, `mount`, identity and network evidence help on NFS; HTTP/API,
IAM, KMS and request logs matter for S3.

### Resume cross-examination warning

The résumé implies enterprise storage and AWS depth. Expect: “Why block rather
than file?”, “Why is S3 not a filesystem?”, and “Which layer owned metadata and
locking?” A definition without a workload decision will sound trained-only.

### Experience boundary

You may explain and lab these models. Do not claim you selected storage for a
real enterprise workload unless you can verify the requirements, alternatives,
your exact decision role and outcome.

---

## 2. NetApp / ONTAP

### Simple explanation and memory hook

ONTAP is NetApp’s storage operating system. It separates physical capacity from
logical data services.

**C-N-A-V-Q/L:** **Cluster → Node → Aggregate → Volume → Qtree or LUN**.
An **SVM** is the logical storage server clients talk to; its data LIFs are the
network endpoints.

```text
ONTAP cluster
  +-- HA pair: node A <---- takeover/giveback ----> node B
        |
        +-- aggregate/pool (RAID-protected physical capacity)
              |
              +-- FlexVol volume
                    +-- NAS namespace / qtree / files
                    +-- SAN LUN (inside the volume)

Client --> data LIF --> SVM --> protocol --> volume/qtree or LUN
```

### How it works

- A **cluster** provides the management and scale-out domain.
- A **node** is a controller/server participating in the cluster.
- An **HA pair** provides controller takeover/giveback capability; HA is not DR.
- An **aggregate** is an ONTAP physical-capacity layer built from RAID-protected
  disks/partitions and owned by a node.
- An **SVM** is a logical storage server with protocol, namespace, security and
  data-access endpoints.
- A **FlexVol volume** is a logical data container backed by an aggregate.
- A **qtree** subdivides a NAS volume for quotas/security/organization.
- A **LUN** is a block object inside a volume and is mapped to hosts through an
  igroup for SAN access.

### Production view

Operators manage capacity/headroom, health, HA, data LIFs and routes,
DNS/AD/identity, export/share policy, LUN mappings, snapshots, replication,
QoS, latency, upgrades and supportability. Versions and platforms matter;
System Manager names and workflows change. Never imply every NetApp platform or
ONTAP release behaves identically.

### 30-second interview answer

> ONTAP provides unified file and block services over a layered architecture.
> RAID-protected capacity is grouped into aggregates; logical volumes consume
> that capacity. SVMs expose isolated data services through LIFs. NAS clients
> access volumes or qtrees through NFS/SMB, while SAN hosts access LUNs inside
> volumes through iSCSI or FC. HA protects controller availability; separate
> replication and recovery design provide DR.

### Senior follow-up

**What survives a controller failure?** Do not promise blindly. Explain that an
HA partner may take over storage and data-service responsibilities, but actual
client impact depends on protocol, multipathing/network design, configuration,
failure type and timeout behavior. Validate from the client, not only the array.

### Troubleshooting angle

Follow client → DNS/route → LIF → SVM/protocol → volume/LUN → aggregate → node.
Correlate client latency with network, protocol, QoS, node/controller and
aggregate/media evidence. An alert is a lead, not root cause.

### Resume cross-examination warning

The résumé says install/configure/manage clustered ONTAP, SVMs, aggregates,
volumes, qtrees and LUNs. An expert will ask platform/model, ONTAP version,
topology, exact commands, what you personally configured, failure behavior and
evidence. These are **UNVERIFIED CLAIMS** until supported.

**REQUIRES REAL EXPERIENCE — DO NOT FABRICATE:** hardware installation,
production takeover/giveback, bank incidents, vendor cases, estate capacity,
cluster counts and upgrade outcomes.

### Experience boundary

Truthful pattern: state whether your depth is training, simulator, lab or
verified work; then draw the architecture and explain the production delta.

---

## 3. NFS / SMB / SAN / iSCSI / FC

### Simple explanation and memory hook

- **SAN = block-storage network architecture**
- **NAS = file-storage architecture/appliance/service**
- **NFS/SMB = file protocols**
- **iSCSI/FC = block transports/protocol stacks**

SAN does not mean “fast.” NAS does not mean “NFS.” NFS is not NAS. FC is not a
filesystem.

| Term | Category | Client receives | Network/dependency |
|---|---|---|---|
| SAN | Architecture | Block devices/LUNs | FC fabric or IP/Ethernet, redundant paths |
| NAS | Architecture/service | Files and directories | Ethernet/IP, identity, file protocol |
| NFS | File protocol | Shared Unix/Linux-oriented namespace | TCP/IP; UID/GID and export policy matter |
| SMB/CIFS | File protocol | Shared Windows-oriented namespace | TCP/IP; AD/Kerberos/DNS/time often matter |
| iSCSI | Block protocol over IP | SCSI LUN | Initiator, IP path, target, mapping |
| Fibre Channel | Block-storage fabric technology | SCSI LUN | HBA, switch fabric, zoning, target ports |

### How it works

```text
BLOCK / iSCSI
Application -> filesystem -> block device -> iSCSI initiator
            -> IP network -> target -> mapped LUN -> controller/media

BLOCK / FC
Application -> filesystem -> block device -> HBA -> FC fabric/zoning
            -> target port -> mapped LUN -> controller/media

FILE / NFS
Application -> VFS/NFS client -> TCP/IP -> NFS server
            -> server filesystem/namespace -> controller/media

FILE / SMB
Application -> SMB client -> DNS/AD/Kerberos + TCP/IP -> SMB server/share
            -> filesystem/ACL -> controller/media
```

### Production view

SAN normally adds zoning, LUN masking/mapping, supported host matrices,
multipathing and coordinated host/storage changes. NAS adds namespaces,
exports/shares, identity, ACLs, locking, client compatibility and file-service
HA. Redundancy must be end to end: two paths through one failed switch are not
two independent paths.

### 30-second interview answer

> SAN and NAS describe storage architectures; NFS, SMB, iSCSI and FC are access
> technologies within those designs. SAN presents block devices, so the host
> usually owns the filesystem. NAS presents files and directories, so the
> server owns the filesystem. iSCSI transports SCSI commands over IP; FC carries
> block traffic through an FC fabric. I troubleshoot each by drawing its actual
> I/O path and checking identity, network/fabric, protocol and storage layers.

### Senior follow-up

**NFS mount is slow—what next?** Define whether mount establishment, metadata,
read or write is slow. Compare clients and time window. Check DNS, route, packet
loss/retransmits, mount options, client CPU/memory, NFS RPC stats, server load,
export/identity, file locking, workload and backend latency. Do not declare
storage guilty from application latency alone.

### Troubleshooting angle

| Symptom | First layers to separate |
|---|---|
| LUN absent | Discovery/login or fabric, zoning, target, host identity, mapping, rescan |
| Paths degraded | HBA/NIC, cable/switch, VLAN/route, target port, MPIO policy |
| NFS permission denied | DNS/path, export policy, client identity, UID/GID, root squash, filesystem mode/ACL |
| SMB auth failure | DNS, time, AD/Kerberos, SPN/account, share ACL, filesystem ACL |
| File service slow | Client workload, metadata/locking, retransmits, server/controller/backend |

### Resume cross-examination warning

“Tier-3 NFS, SMB/CIFS and SAN support” implies real incidents. Expect exact
commands, counters, timeline, scope, mitigation and prevention—not protocol
definitions.

**REQUIRES REAL EXPERIENCE — DO NOT FABRICATE:** “most common incident,” real
business impact, escalation, ticket, outage and root-cause story.

### Experience boundary

If labbed, say what you built and broke. Then give a hypothetical production
method. Do not call a Samba/NFS/iSCSI lab Tier-3 enterprise support.

---

## 4. LUN / SVM / aggregate / volume / qtree

### Simple explanation and memory hook

Think **capacity below, service above**:

```text
Physical media -> RAID -> pool/aggregate -> volume
                                           +-> qtree -> files/share/export
                                           +-> LUN -> host filesystem -> files

SVM + data LIFs = logical server and access doorway
```

| Object | Vendor-independent meaning | ONTAP mapping/caution |
|---|---|---|
| Physical media | SSD/HDD/device capacity | Disks or partitions; platform-dependent |
| RAID group | Failure protection set | Used within aggregate; RAID type matters |
| Pool | Capacity/placement layer | Aggregate is ONTAP’s relevant physical layer |
| Volume | Ambiguous logical container | FlexVol/FlexGroup; ask what “volume” means |
| LUN | Logical block device presented to hosts | Lives in a volume; mapped via igroup |
| Filesystem | Organizes blocks into files/metadata | Host-owned on LUN; ONTAP-owned for NAS |
| Mount point | Client path attaching filesystem | `/data`, drive mapping, namespace junction |
| Export/share | Policy/name exposing files | NFS export policy or SMB share |
| SVM | Logical storage server | Owns protocols, endpoints and namespace context |
| Qtree | Subdivision within a NAS volume | Useful for quotas/security/organization; not a volume |

### How it works

**SAN path:** aggregate backs volume; volume contains LUN; host identity is
authorized; host discovers/maps LUN; OS sees a block device; host creates and
mounts filesystem.

**NAS path:** aggregate backs volume; storage system owns filesystem/namespace;
SVM exposes it via NFS export or SMB share; client mounts/connects it.

### Production view

Provisioning includes naming, ownership, capacity headroom, thin-provisioning
risk, host compatibility, access control, protection, monitoring and safe
resize. A 10 TiB thin volume does not mean 10 TiB is physically reserved.
Filesystem expansion is a separate host step after a block device grows.

### 30-second interview answer

> An aggregate is the ONTAP physical-capacity and RAID placement layer. A volume
> is a logical data container backed by that aggregate. A LUN is a block object
> inside a volume and is mapped to a host; the host then owns the filesystem. A
> qtree subdivides a NAS volume for organization, quotas or security. An SVM is
> the logical storage server exposing protocols and data endpoints. These are
> different layers, not synonyms.

### Senior follow-up

**LUN resized but filesystem unchanged—why?** Storage growth only enlarged the
LUN. The host may need a rescan, multipath/device resize, partition/LVM growth
and filesystem expansion. Each step has its own validation and rollback risk.

### Troubleshooting angle

Walk downward until evidence changes:

1. application/file error;
2. mount/filesystem health and free bytes/inodes;
3. block device or NFS/SMB client;
4. mapping/export/share and identity;
5. network/fabric/path;
6. LIF/target/SVM;
7. volume/QoS/capacity;
8. aggregate/controller/media.

### Resume cross-examination warning

Expect “What exactly did you configure?”, “Why qtree rather than volume?”,
“What is the SVM data path?”, and “How did the host see the LUN?” The résumé
also claims aggregate creation, which can imply physical RAID operations.

**REQUIRES REAL EXPERIENCE — DO NOT FABRICATE:** production aggregate builds,
host coordination, capacity scale, change approvals and incidents.

### Experience boundary

Explain the generic layers and any lab. Name ONTAP versions/platforms only if
verified; do not invent commands or physical topology from memory.

---

## 5. Snapshots / SnapMirror / SnapVault

### Simple explanation and memory hook

- **Snapshot = point-in-time reference/copy within a system**
- **SnapMirror = replicate for protection/mobility/DR policies**
- **SnapVault = retention-oriented backup-style protection term/workflow; be
  version-aware because modern ONTAP unifies much protection under SnapMirror**

**Snapshot is not automatically backup. Replication is not automatically DR.**

### How it works

```text
Primary RW volume -- local snapshots
       |
       +-- scheduled/asynchronous or supported synchronous replication
       v
Destination DP volume (normally read-only until a planned break/failover)
```

Snapshot efficiency commonly relies on shared unchanged blocks. Changed blocks
increase snapshot space consumption. Crash-consistent does not automatically
mean application-consistent.

SnapMirror asynchronous protection transfers snapshots/changed blocks according
to policy/schedule. Relationship type and ONTAP version matter. A destination
must be made writable to serve data in many DR workflows; direction must then be
handled carefully for resync/failback.

### Production view

Define retention, RPO, consistency, bandwidth, lag alert, destination security,
dependency order, DNS/client cutover, test schedule and failback. Monitor
relationship health and last successful transfer—not only job existence.

### 30-second interview answer

> A snapshot gives a point-in-time recovery reference but usually depends on the
> source storage, so it is not by itself an independent backup. SnapMirror
> replicates protected data to another destination under a defined policy and
> is commonly used for DR or data protection. SnapVault historically describes
> retention-oriented backup relationships; I would confirm the ONTAP version
> and policy because terminology evolved. A complete design still needs tested
> restore, failover, failback, RPO/RTO and operational monitoring.

### Senior follow-up: failover/failback

Conceptual sequence—verify exact commands against the product/version and
runbook:

1. confirm outage, authority and last common recovery point;
2. stop/quiesce source writes if possible and perform final update if safe;
3. verify destination consistency and relationship state;
4. break/promote destination to writable;
5. redirect clients/DNS/application dependencies;
6. validate data and service, monitor and record divergence;
7. repair original source;
8. reverse-resynchronize from the currently active site to the original source,
   understanding that resync can discard destination-side changes outside the
   chosen common point;
9. plan a controlled cutback, reverse direction again as required, validate and
   restore protection.

Never say “just break and resync.” Data direction and authoritative copy are the
central risks.

### Troubleshooting angle

For lag/failure check relationship state, last successful update, common
snapshot, source changes, schedule/policy, intercluster network, peers, capacity,
permissions, throttles and error events. Protect the known-good copy before
destructive resync/reinitialize decisions.

### Resume cross-examination warning

The résumé claims configuration and troubleshooting of SnapMirror/SnapVault and
DR. Expect exact relationship type, policy, schedule, lag, monitoring, break,
resync, reverse direction, restore and failure evidence.

**REQUIRES REAL EXPERIENCE — DO NOT FABRICATE:** real failover/failback, bank DR
test, missed RPO, data loss or replication incident.

### Experience boundary

You can give the engineering sequence and label it conceptual/lab. Personal DR
questions require verified evidence; do not manufacture a test story.

---

## 6. Storage performance

### Simple explanation and memory hook

**IOPS = HOW MANY. THROUGHPUT = HOW MUCH. LATENCY = HOW LONG. QUEUE DEPTH = HOW
MANY WAITING/OUTSTANDING. I/O SIZE = HOW BIG EACH ONE IS.**

| Measure | Unit | Meaning | Common trap |
|---|---|---|---|
| IOPS | operations/s | Completed I/O count per second | High IOPS may be demand, not health |
| Latency | µs or ms | Time for an operation | Average hides tail latency |
| Throughput | MB/s or MiB/s | Data transferred per second | Confusing bytes with bits |
| Queue depth | count | Outstanding/waiting I/O at a layer | Symptom, tuning input and possible cause |
| I/O size | KiB/MiB | Bytes per operation | Workload averages hide a distribution |

Approximation:

```text
Throughput ≈ IOPS × average I/O size
```

Example using binary units:

- 10,000 IOPS × 4 KiB ≈ 39.1 MiB/s.
- 1,000 IOPS × 1 MiB ≈ 1,000 MiB/s.

The second workload has one tenth the IOPS but about 25.6 times the throughput.
Real results diverge because of protocol overhead, caching, compression/dedupe,
read/write mix, concurrency, device/network limits and measurement boundaries.

### How it works

Random small I/O stresses IOPS and seek/translation/metadata paths. Large
sequential I/O stresses bandwidth. Writes may require protection work and
durability acknowledgment. Cache hits can make backend media look quiet; cache
misses or destaging can reveal limits. Burst performance is temporary;
baselines describe sustainable normal behavior.

Percentiles matter: p99 = 99% completed at or below that latency; the slowest 1%
were worse. It does not mean “99% availability.”

### Production view

Describe the full workload profile: operation, size distribution, random/
sequential, read/write ratio, concurrency, working set, cache state, time,
growth and SLO. Compare same time window across layers. Watch saturation knees:
throughput/IOPS flatten while queue and latency rise.

### 30-second interview answer

> I never assess storage from IOPS alone. I define the workload by I/O size,
> read/write mix, random/sequential pattern and concurrency. I correlate IOPS,
> throughput, latency percentiles and queue depth against a baseline and service
> limits. High IOPS can be healthy demand; high latency can originate in the
> application, filesystem, protocol, network, controller, cache or media. The
> key is finding where response time is introduced.

### Senior follow-up

**High IOPS but application remains slow—why?** Possibilities: application lock
or CPU delay; many small inefficient I/Os; high tail latency; queueing; sync
writes; network retransmits; metadata/locking; hot object/LUN; cache miss;
downstream saturation; or metrics measured at different layers/times. Trace an
application request and correlate percentiles, not just totals.

### Troubleshooting angle: latency decomposition

```text
Application wait/locks
        |
OS/filesystem/page cache
        |
Protocol/client/driver/multipath
        |
Network/fabric: DNS, loss, retransmit, congestion
        |
Storage frontend/LIF/target/QoS
        |
Controller CPU/cache/HA work
        |
Pool/aggregate/media
```

Reusable 13 steps:

1. Define symptom. 2. Scope. 3. Timeline. 4. Recent changes. 5. Baseline.
6. Follow I/O path. 7. Measure each layer. 8. Form hypotheses. 9. Test one at a
time. 10. Mitigate safely. 11. Validate recovery. 12. Determine root cause.
13. Prevent recurrence.

Low CPU does not prove storage health: a process may be blocked waiting for I/O,
locks or network. High queue depth may result from demand exceeding service
rate; increasing it can improve parallelism until saturation, then worsen
latency.

### Resume cross-examination warning

The résumé says performance diagnostics identified bottlenecks. Expect a real
counter, baseline, layer, hypothesis, change and result. Generic “check CPU,
memory and disk” will not survive.

**REQUIRES REAL EXPERIENCE — DO NOT FABRICATE:** a historical bottleneck,
customer impact, before/after metric or RCA.

### Experience boundary

Truthfully describe `fio`/local lab results as lab results. The engineering
method transfers; the workload and production outcome do not.

---

## 7. RAID / HA / multipathing

### Simple explanation and memory hook

- **RAID protects data across media failures.**
- **HA keeps a local service available through component/controller failure.**
- **Multipathing keeps host-to-storage access through path failure.**
- **DR recovers from site/region or broader failure.**

None is a substitute for backup.

### How it works

| Mechanism | Protects primarily against | Does not automatically protect against |
|---|---|---|
| RAID | One or more media failures, per layout | Deletion, corruption, array/site loss |
| Controller HA | Controller/server failure | Site loss, common dependency, bad change |
| Multipathing | HBA/NIC/switch/target/path failure | Array/site failure; correlated paths |
| Replication | Source-system/site failure | Logical corruption/deletion if replicated |
| Backup | Recovery points and retention | Instant continuity unless designed for it |

Multipath software combines multiple OS-visible paths into one logical device.
ALUA/optimized-path knowledge helps select appropriate paths on asymmetric
arrays. Queueing policies during total path loss can affect whether applications
wait or fail.

### Production view

Validate independence: separate fabrics/switches/ports/power where required.
Use supported host/firmware/driver matrices. Monitor degraded states; redundancy
silently lost today becomes outage during tomorrow’s maintenance.

### 30-second interview answer

> RAID, HA and multipathing protect different layers. RAID protects data across
> media failures, controller HA protects service through a controller failure,
> and multipathing protects host access through path failures. I design them as
> independent layers, test failover from the client, monitor degraded states and
> still maintain backup and DR because none protects against every failure.

### Senior follow-up

**Two paths are visible—is the design resilient?** Not necessarily. Both may
share one NIC, switch, fabric, target, controller, power feed or route. Draw both
paths end to end and test one component failure at a time.

### Troubleshooting angle

Separate device failure from path failure. Check OS multipath state, path error
counters, HBA/NIC, switch/fabric/network, target ports, controller ownership,
ALUA state and recent zoning/mapping/change. Avoid removing devices or forcing
failover without confirming the exact target and protection state.

### Resume cross-examination warning

SAN Tier-3 and aggregate claims imply RAID, path and HA incidents. Production
takeover, rebuild and fabric failures require real evidence.

### Experience boundary

State if your failover was virtual/iSCSI/simulator-based. Do not convert it into
FC fabric, hardware or production HA ownership.

---

## 8. Backup / replication / DR

### Simple explanation and memory hook

- **Backup = retained recovery copy**
- **Replication = keep another copy current**
- **DR = people + process + technology to restore business service**

### How it works

```text
Production data
   +-- snapshots --------> fast local recovery
   +-- backup -----------> retained/isolated recovery points
   +-- replication ------> secondary system/site
                                      |
                         runbook + apps + DNS + identity
                                      |
                                      v
                                  DR service
```

Replication can quickly reproduce deletion, encryption or corruption. Backup
needs restore testing. DR needs dependency orchestration and user validation.

### Production view

Use workload tiers. Define backup frequency, consistency, retention, immutable/
isolated copy, encryption/key recovery, replication mode, bandwidth, dependency
order, authority to declare, communications, exercises and failback. “Job
succeeded” is not “recovery works.”

### 30-second interview answer

> Backup provides retained recovery points, replication maintains another copy,
> and DR restores an end-to-end business service. A resilient design combines
> them according to RPO/RTO, protects copies from the same failure or attacker,
> monitors protection gaps, and repeatedly proves restore, failover and failback
> with application-level validation.

### Senior follow-up

**Backup success is green; restore fails. Why?** Missing dependencies, expired/
corrupt catalog, KMS/key or permission loss, incompatible target, incomplete
application consistency, unavailable network/DNS/identity, capacity shortage or
untested procedure. Preserve evidence and prioritize a safe alternate recovery
point.

### Troubleshooting angle

Check last usable recovery point, not last job. Verify source selection, policy,
schedule, agent/service, destination capacity, credentials/KMS, network,
retention, catalog and restore prerequisites. Test restored data/application.

### Resume cross-examination warning

Résumé language about backup and DR causes interviewers to expect restore tests,
failed jobs, RPO/RTO and evidence. Do not treat snapshots as full backup.

### Experience boundary

**REQUIRES REAL EXPERIENCE — DO NOT FABRICATE:** production restore, DR
exercise, audit evidence, outage or actual achieved RPO/RTO.

---

## 9. RPO / RTO / failover / failback

### Simple explanation and memory hook

- **RPO = DATA LOSS WINDOW**
- **RTO = RECOVERY TIME**
- **Failover = move service to recovery location**
- **Failback = safely return service and protection direction**

### How it works

If failure occurs at 10:00 and latest usable recovery point is 09:45, actual
data exposure is 15 minutes. If service is validated for users at 10:28,
recovery took 28 minutes. Be explicit about whether detection and decision time
are included; business RTO normally concerns service restoration, not merely
storage promotion.

### Production view

RPO/RTO come from business impact and dependency reality. A 15-minute snapshot
schedule does not guarantee 15-minute achieved RPO if jobs lag or recovery
points are unusable. A storage failover in five minutes does not meet a 30-minute
application RTO if DNS, identity, databases or validation take an hour.

### 30-second interview answer

> RPO is the maximum tolerable data-loss window; RTO is the maximum tolerable
> time to restore usable service. I translate them into protection frequency,
> replication mode, recovery architecture and runbooks, then measure achieved
> RPO/RTO through tests. Failover makes the recovery site authoritative;
> failback reconciles changes, reverses protection safely, cuts service back and
> revalidates it.

### Senior follow-up

For RPO 15 min/RTO 30 min, demand dependency inventory and measurable tests.
Use replication frequent enough to leave margin, independent backup for
corruption, pre-provisioned or rapidly provisioned compute/network/identity,
automated ordered recovery, DNS/connection strategy and rehearsed validation.
State cost and operational trade-offs.

### Troubleshooting angle

When RPO is missed: verify job/relationship, lag calculation, change rate,
bandwidth, throttles, capacity, common snapshots, errors and alerts. When RTO is
missed: break elapsed time into detect, decide, data, infrastructure,
application, dependency and validation stages.

### Resume cross-examination warning and boundary

Do not claim you negotiated or achieved production RPO/RTO without evidence.
Give a design and say it is a hypothetical approach when that is the truth.

---

## 10. AWS storage: S3, EBS, EFS, FSx, FSx for ONTAP

### Simple explanation and memory hook

| Service | Model | Best mental fit | Main caution |
|---|---|---|---|
| S3 | Object | API-scale durable objects | Not a normal POSIX filesystem |
| EBS | Block | EC2/EKS host-attached volume | AZ and attachment/topology matter |
| EFS | Managed NFS file | Shared Linux file access | NFS semantics, identity and throughput model |
| FSx for Windows | Managed Windows file | SMB/AD workloads | AD/DNS and Windows semantics |
| FSx for Lustre | Managed parallel file | HPC/ML high-throughput file | Specialized workload/operations |
| FSx for OpenZFS | Managed OpenZFS file | ZFS/NFS-oriented workloads | Match protocol/features carefully |
| FSx for ONTAP | Managed ONTAP file/block | NFS/SMB/iSCSI and ONTAP features | Network, capacity/throughput and shared responsibility |

### How it works

S3 stores objects in buckets accessed via API. EBS exposes a block volume to
compute. EFS exposes regional managed NFS using mount targets. FSx offers
managed file systems aligned to specific workload ecosystems. FSx for ONTAP
contains file-system infrastructure, SVMs and volumes; clients reach SVM
endpoints using supported NAS/SAN protocols.

### Production view

Choose on semantics and workload, then examine availability, failure domains,
performance dimensions, identity/network, encryption, backup/DR, quotas and
cost. Managed does not mean “no operations”: capacity, access, telemetry,
recovery and application behavior remain yours.

**EFS vs FSx for ONTAP:** EFS is the simpler AWS-managed NFS choice for many
Linux shared-file workloads. FSxN fits when ONTAP protocols/features,
multi-protocol needs, snapshots/SnapMirror, storage efficiencies, tiering or
migration compatibility justify greater design/operational complexity.

### 30-second interview answer

> S3 is object, EBS is block, and EFS/FSx are managed file-system families.
> I choose S3 for object-native scale and lifecycle, EBS for block semantics
> attached to compute, EFS for straightforward shared Linux NFS, and an FSx
> family when the workload needs that filesystem ecosystem. FSx for ONTAP is a
> strong hybrid option for NFS/SMB/iSCSI and ONTAP features, but I still validate
> network, HA deployment, performance, backup, security and cost.

### Senior follow-up

**S3 vs EFS vs EBS for a shared application?** Ask whether the app can use an
object API. If yes, S3 may remove filesystem scaling concerns. If it requires
shared POSIX-like NFS, evaluate EFS. If it needs block semantics and sharing is
not required or handled by a cluster filesystem/application, evaluate EBS.

### Troubleshooting angle

- S3: endpoint/DNS, IAM/bucket policy, explicit deny, KMS, object key/version,
  request throttling/error and client behavior.
- EBS: attachment/AZ, OS device, filesystem, volume limits, queue/latency,
  snapshot/KMS.
- EFS: DNS/mount target, route/SG, NFS port, mount options, UID/GID/access point,
  throughput and client/server evidence.
- FSxN: endpoint/routes/SG, SVM/protocol identity, volume/capacity/tiering,
  throughput/IOPS, KMS, HA/service events.

### Resume cross-examination warning

FSx ONTAP is explicitly claimed. Expect deployment type, HA, VPC/endpoints,
SVMs, protocols, SSD/capacity pool, tiering, throughput, backup/SnapMirror and
exact personal actions.

**REQUIRES REAL EXPERIENCE — DO NOT FABRICATE:** implemented/managed production
FSxN, scale, migration, incident or savings.

### Experience boundary

A comparison is conceptual. Console walkthrough is training. A retained sandbox
deployment is lab-practiced. None proves production ownership.

---

## 11. AWS Backup / DataSync / Storage Gateway

### Simple explanation and memory hook

- **Backup = protect and retain**
- **DataSync = move/copy online data**
- **Storage Gateway = bridge on-prem applications to AWS storage through a
  local appliance/cache**

| Service | Primary job | Not the same as |
|---|---|---|
| AWS Backup | Central policy/vault-based backup across supported resources | HA or instant DR by itself |
| DataSync | Managed data transfer between supported locations | A permanent file-service frontend |
| Storage Gateway | Hybrid storage interface with local appliance behavior | Bulk migration-only tool |

### How it works / production view

AWS Backup applies plans/selections to resources and stores recovery points in
vaults; design KMS, retention, account/Region isolation and restore tests.
DataSync uses locations, tasks and often an agent for on-prem paths; design
bandwidth, filters, metadata, verification, incremental waves and cutover.
Storage Gateway modes provide file, volume or tape integration; operate the
appliance, cache, credentials, network and upload/backlog behavior.

### 30-second interview answer

> I use AWS Backup for policy-driven protection and tested restores, DataSync
> for controlled online transfer and migration, and Storage Gateway when an
> on-prem application needs a continuing hybrid interface to AWS storage. They
> solve different lifecycle problems. In production I add KMS and permissions,
> bandwidth/capacity planning, monitoring, recovery validation and failure
> procedures.

### Senior follow-up and troubleshooting

Large NFS migration: inventory data/metadata and change rate; choose target;
place/secure agent; seed data; run incremental syncs; monitor errors/bandwidth;
validate counts/checksums/metadata/application; freeze writes; final sync;
redirect clients; observe; retain rollback; decommission later.

For failures, check agent/appliance health, source/target reachability,
permissions/KMS, filters/path, metadata compatibility, destination capacity,
bandwidth/throttles, task logs and validation discrepancies.

### Resume warning and boundary

Cloud/migration claims may trigger “what was the data size and cutover?” Those
historical facts **REQUIRE REAL EXPERIENCE — DO NOT FABRICATE**. Give a clearly
hypothetical runbook if not verified.

---

## 12. IAM / KMS / VPC / Route 53

### Simple explanation and memory hook

**NAME → PATH → IDENTITY → KEY**

- Route 53/DNS answers **where/name**.
- VPC networking provides **path**.
- IAM decides **who may call what**.
- KMS controls use of the **encryption key**.

```text
Client -> DNS/Route 53 -> route/SG/NACL/endpoint -> AWS service
          identity/role -> IAM + resource policy -> KMS policy/grant
```

### How it works

IAM evaluation combines identity and resource policies plus possible boundaries,
session policies and organization controls; explicit deny wins. KMS authorization
has its own key-policy/grant context in addition to caller permissions. VPC path
depends on address, route tables, SGs, NACLs, endpoints, return route and DNS.
Route 53 provides public/private hosted zones, routing policies and resolver
integration; TTL affects change propagation/caching.

### Production view

Use roles and short-lived credentials, least privilege, private paths where
appropriate, auditable key ownership, resilient DNS/connectivity and logs such
as CloudTrail/flow/service logs. Avoid “open SG to test” without bounded scope
and immediate rollback.

### 30-second interview answer

> For an AWS storage access failure I separate name resolution, network path,
> identity authorization and encryption-key authorization. I verify the caller
> and request, DNS result, routes/security controls/endpoints, IAM and resource
> policies, then KMS key policy or grants and audit events. An IAM allow alone
> does not guarantee success if another layer explicitly denies the request.

### Senior follow-up

**IAM allows S3 but KMS denies:** confirm actual principal/session, encryption
context/key, IAM action permissions, key policy/grant, cross-account design,
resource/bucket policy, endpoint policy and explicit denies. Use CloudTrail and
the service error to test one hypothesis; do not weaken all policies.

### Troubleshooting angle

Trace packet/request both directions. Distinguish timeout (often path) from
authenticated `AccessDenied` (policy/key) while recognizing services vary.
Check DNS and system time before deep authentication debugging.

### Resume warning and boundary

AWS proficiency and Route 53/IAM claims invite cross-account, endpoint, policy
evaluation and real incident questions. Do not claim enterprise IAM governance
or hybrid networking ownership without evidence.

---

## 13. Terraform

### Simple explanation and memory hook

**CONFIG says desired; STATE maps real; PLAN explains change; APPLY executes.**

```text
HCL configuration + variables
          |
       provider API <----> real infrastructure
          |
         state = resource identity and known attributes
          |
      plan -> review/gate -> apply -> updated state
```

### How it works

Terraform builds a dependency graph, asks providers to read/change APIs and
records resource mappings in state. Modules create reusable contracts. Remote
state enables team workflows; locking prevents concurrent writes where the
backend supports it. `for_each` keys should be stable. Lifecycle rules and
explicit dependencies are precision tools, not fixes for poor design.

### Production view

Separate environments/accounts and state blast radii; encrypt and restrict
state; pin/test versions; review saved plans; use short-lived pipeline identity;
scan/test policy; back up state; prohibit routine manual change; document import,
moved resources and emergency recovery.

Terraform does **not** automatically roll back a partially completed apply.
After an error, inspect what exists and current state, fix the cause, plan again
and reconcile safely.

### 30-second interview answer

> Terraform is declarative IaC. Configuration defines desired infrastructure,
> providers call platform APIs, and state maps resource addresses to real
> objects. In production I use a secured remote backend with locking, small
> state blast radii, versioned modules, reviewed plans, CI gates and controlled
> identities. For drift or a partial apply, I preserve state, inspect reality,
> fix the cause, re-plan and reconcile rather than editing or deleting blindly.

### Senior follow-up

**State corruption:** stop applies; preserve backend versions/backups and pull a
copy; identify last trusted state and compare it with actual infrastructure;
recover through backend/versioned state or documented state operations; verify
serial/lineage and plan carefully. `force-unlock` is only for a confirmed stale
lock, not an active run. Manual `state push` is high risk and requires backup,
review and exact workspace confirmation.

**Partial apply:** read error; inventory created/changed resources and state;
fix quota/permission/input/provider problem; run fresh plan; import only if a
real object exists outside state; apply to converge; validate and record.

### Troubleshooting angle

Classify: syntax/config, dependency, provider/auth, API/quota, lock/backend,
drift/state, replacement or pipeline identity. Never delete state to “start
over” in production.

### Resume cross-examination warning

The résumé claims reusable modules across dev/QA/prod. Expect module interface,
versioning, state separation, backend/lock, testing, consumers, breaking change,
drift and failed apply.

**REQUIRES REAL EXPERIENCE — DO NOT FABRICATE:** production state incident,
consumer count, environment ownership and recovery outcome.

### Experience boundary

Show a real lab repo if available and call it a lab. A tutorial module is not
organizational platform ownership.

---

## 14. GitHub Actions / CI-CD

### Simple explanation and memory hook

**CHECK → PLAN → APPROVE → APPLY → VERIFY → ROLLBACK/RECOVER**

### How it works

GitHub Actions runs YAML workflows triggered by repository events or schedules.
Jobs run on hosted or self-hosted runners; permissions, environments, secrets/
OIDC, reusable workflows, artifacts and concurrency control shape security and
delivery behavior.

```text
PR -> lint/test/security -> terraform plan -> human/environment gate
   -> protected apply -> smoke/health validation -> evidence
```

### Production view

Pin trusted actions, minimize `GITHUB_TOKEN` and cloud permissions, prefer
short-lived federation, protect environments, separate plan/apply, preserve the
reviewed artifact, serialize state-changing runs, audit releases and define
recovery. Rollback for infrastructure is not always “apply old code”; data and
irreversible changes require forward recovery or explicit runbooks.

### 30-second interview answer

> A safe infrastructure pipeline validates code, produces a reviewed plan,
> enforces security and policy gates, uses short-lived least-privilege identity,
> protects the target environment, serializes state-changing runs, applies the
> approved change and validates service afterward. I design explicit recovery
> because CI/CD success and deployment success are not the same.

### Senior follow-up / troubleshooting

For failure, locate trigger/filter, YAML/expression, runner, dependency/cache,
permissions/OIDC trust, secret, environment approval, artifact, Terraform lock/
state or target service. Preserve logs, avoid blind reruns after partial change,
inspect actual target, then recover safely.

### Resume warning and boundary

“Centralized pipelines” and 25–30% improvements imply platform consumers,
runner architecture, governance and metric calculation.

**REQUIRES REAL EXPERIENCE — DO NOT FABRICATE:** production ownership, failed
deployment, stakeholder adoption, cost or speed percentage. A lab workflow can
prove mechanics only.

---

## 15. Kubernetes / EKS storage

### Simple explanation and memory hook

**PVC asks; StorageClass provisions; PV represents; CSI connects.**

```text
Pod -> PVC -> PV -> CSI driver -> EBS/EFS/other storage
          ^
     StorageClass + topology + access mode
```

### How it works

A PVC requests capacity/access properties. A StorageClass defines dynamic
provisioning behavior. A PV represents storage bound to a claim. The CSI
controller provisions/attaches; the node component mounts. Access mode and
topology must match the workload. EBS is typically AZ-scoped block storage;
EFS provides shared NFS behavior.

### Production view

Operate CSI add-ons, IAM, encryption, topology, reclaim policy, expansion,
snapshots/backup, quotas, node/AZ failure, stateful application consistency and
upgrade compatibility. A StatefulSet provides stable identities/orchestration;
it does not make data protected automatically.

### 30-second interview answer

> Kubernetes separates a workload’s storage request from the underlying
> implementation. The PVC requests storage, a StorageClass drives dynamic
> provisioning, the PV represents the result, and CSI components provision,
> attach and mount it. On EKS I check access mode, AZ topology, CSI/IAM, node
> attachment and mount behavior, then add snapshots/backup and application-
> consistent recovery separately.

### Senior follow-up

**PVC Pending:** inspect PVC events and StorageClass; provisioner/CSI controller;
parameters, quota/capacity and IAM; binding mode; requested access mode; allowed
topologies/AZ; pod scheduling and node labels. Do not delete the claim first.

**Zero-downtime EKS upgrade:** inventory versions/add-ons/APIs; validate skew and
compatibility; test; upgrade control plane; update CNI/CoreDNS/kube-proxy/CSI;
replace node groups gradually with spare capacity; honor PDBs/readiness; monitor
user SLI; stop on error. Rollback constraints must be known—do not promise a
simple control-plane downgrade.

### Troubleshooting angle

Pending = provision/bind/topology; attachment errors = volume/AZ/CSI/IAM/stale
attachment; mount errors = filesystem, device, NFS/network, permissions, node
plugin; app errors = path/ownership/locking/consistency.

### Resume warning and boundary

Zero-downtime EKS upgrades and migration outcomes are Extreme-risk claims.

**REQUIRES REAL EXPERIENCE — DO NOT FABRICATE:** exact production versions,
clusters, observed zero downtime, incident, 40% improvement or stakeholder
role. Describe a lab upgrade as lab-practiced.

---

## 16. Prometheus / Grafana

### Simple explanation and memory hook

**Prometheus collects and queries metrics; Grafana makes them understandable.**

**Metric → baseline → symptom → hypothesis → action → validation.**

### How it works

Prometheus commonly scrapes labeled time series and evaluates PromQL/recording/
alert rules. Grafana queries data sources for dashboards and alert context.
Labels enable slicing but uncontrolled cardinality increases cost and risk.

### Production view

Define SLIs/SLOs and ownership before dashboards. Include application latency/
errors, host/resource saturation, network/protocol, storage latency/IOPS/
throughput/queue/capacity and protection health. Alerts must be actionable and
linked to a runbook; missing telemetry is itself a condition to monitor.

### 30-second interview answer

> Prometheus provides labeled time-series collection, PromQL and rule
> evaluation; Grafana provides dashboards and operational context. For storage I
> correlate application latency with client, network/protocol, controller,
> volume/pool and media signals using the same time window and a baseline. A
> dashboard supports a diagnosis—it does not prove root cause or create 99.9%
> uptime by itself.

### Senior follow-up / troubleshooting

If a graph is empty: check target discovery, DNS/network, endpoint, auth/TLS,
scrape health, labels/query/time range and retention. If an alert fires: verify
user impact and data validity, correlate layers, form hypotheses, mitigate and
validate. Avoid averaging away p95/p99 symptoms.

### Resume warning and boundary

The résumé connects these tools to 99.9% uptime and performance improvement.
Those outcomes require SLI definition, period, calculation, causality and real
evidence.

**REQUIRES REAL EXPERIENCE — DO NOT FABRICATE:** historical uptime, incident
prevention, dashboard impact or production RCA.

---

## Final oral drill

Answer these without notes:

1. Block vs file vs object—and EBS vs EFS vs S3?
2. SAN vs NAS; iSCSI vs FC; NFS vs SMB?
3. Aggregate vs volume vs LUN vs qtree vs SVM?
4. Walk the host-to-LUN path and the NFS path.
5. Snapshots vs backup vs SnapMirror; conceptual failover/failback?
6. High IOPS but slow application—what hypotheses and evidence?
7. RAID vs HA vs multipathing vs DR?
8. Design RPO 15 minutes/RTO 30 minutes.
9. EFS vs FSx for ONTAP?
10. Recover from Terraform partial apply/state corruption?
11. Troubleshoot PVC Pending?
12. How do you state a training/lab boundary without bluffing?

## Authoritative references for mutable behavior

Use these after the interview for version-specific commands and limits; tonight,
focus on principles:

- [NetApp: resynchronize a SnapMirror relationship](https://docs.netapp.com/us-en/ontap/data-protection/resynchronize-relationship-task.html)
- [AWS: how FSx for ONTAP works](https://docs.aws.amazon.com/fsx/latest/ONTAPGuide/how-it-works-fsx-ontap.html)
- [HashiCorp: recover Terraform state](https://developer.hashicorp.com/terraform/cli/state/recover)
- [HashiCorp: errors during apply](https://developer.hashicorp.com/terraform/tutorials/cli/apply#errors-during-apply)
- [Kubernetes: persistent volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
