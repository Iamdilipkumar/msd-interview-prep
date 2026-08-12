# 04 — S3 Deep Dive

## Mental model

S3 is a regional object store: a bucket contains objects addressed by keys. It is not a mounted block device or a general POSIX filesystem. Design around API calls, object replacement, request rates, and lifecycle.

| Concern | Design choice |
|---|---|
| Access | IAM/access point/bucket policy; Block Public Access; VPC endpoint policy |
| Protection | Versioning, Object Lock, replication, backup where required |
| Encryption | SSE-S3 or SSE-KMS; manage KMS permissions and quotas |
| Cost | Storage class, request volume, retrieval, lifecycle, transfer |
| Performance | Parallel requests, multipart upload, byte-range fetch, edge caching |

S3 provides strong read-after-write consistency for object PUT/DELETE and listing operations. Distributed application races still exist: two writers can overwrite the same key; use conditional requests or an external coordination design where required.

### Buckets, objects, keys, and prefixes

A bucket is a regional namespace and policy/management boundary. An object is data plus system/user metadata and is addressed by a complete key. “Folders” are console presentation of shared key prefixes; they are not real directories with POSIX rename/locking semantics. Design key names for listing, lifecycle, events, access controls and analytics—not for an imagined filesystem.

## Storage classes

| Pattern | Consider | Watch |
|---|---|---|
| Frequent, unpredictable | S3 Standard or Intelligent-Tiering | Monitoring/automation charge for Intelligent-Tiering |
| Infrequent, millisecond | Standard-IA / One Zone-IA | Minimum duration/size and retrieval charges |
| Archive | Glacier Instant/Flexible/Deep Archive | Retrieval time and restore workflow |

Lifecycle rules transition or expire objects and noncurrent versions. Validate overlapping rules, minimum storage durations, delete markers, and replication interactions.

## Versioning, replication, and Object Lock

- Versioning makes overwrite/delete recovery possible but does not remove versions automatically.
- Replication is asynchronous; define destination ownership, KMS permissions, replica modification behavior, metrics, and failback.
- Object Lock uses retention/legal holds for WORM requirements and requires a versioned bucket. Governance and compliance modes have different override semantics.
- Replication is not a substitute for logically isolated recovery when deletion, corruption, or compromised credentials can propagate.

**SRR** copies within a Region; **CRR** copies to another Region. Both can support ownership/compliance/latency/recovery requirements. Define scope, replication IAM role, destination policy/ownership, KMS grants, storage class, metrics, replication of delete markers and whether existing objects need Batch Replication.

Object Lock retention prevents deletion/overwrite of protected versions until a date; a legal hold has no expiry and is explicitly removed. Governance mode can be bypassed by specially authorized principals; compliance mode is stricter. Treat retention configuration as a legal and operational decision.

## Encryption and access-control layers

| Control | What it does | Trap |
|---|---|---|
| IAM policy | Grants caller operations | Correct bucket/object ARN still required |
| Bucket policy | Resource-level principals/conditions | Explicit deny can block all other allows |
| ACL | Legacy object/bucket grants | Prefer Bucket owner enforced Object Ownership where possible |
| Block Public Access | Guardrail against public policy/ACL patterns | It is not a complete least-privilege policy |
| SSE-S3 | S3-managed keys | Less customer policy separation |
| SSE-KMS | KMS-backed envelope encryption | Adds KMS policy, quota and request-cost dependency |

S3 static website endpoints serve website behavior such as index/error documents but do not provide HTTPS themselves. A secure modern public site commonly uses a private S3 origin behind CloudFront with Origin Access Control, TLS and optional WAF rather than making the bucket public.

## Multipart upload, presigned URLs, events, and access points

- **Multipart:** initiate, upload numbered parts independently, then complete; use checksums and abort incomplete uploads. It enables parallelism/resume for large objects.
- **Presigned URL:** time-limited signed API request carrying the signer’s authority. Limit method/key/expiry and treat the URL as sensitive while valid.
- **Event notifications:** send selected object events to supported destinations. Consumers must tolerate duplicates, ordering differences and retries; use idempotency and reconciliation.
- **Access point:** named S3 access endpoint/policy for a specific application or network origin. It reduces giant shared bucket policies but does not bypass bucket, IAM, KMS or Block Public Access controls.

## Access troubleshooting

```text
caller session -> identity policy -> SCP/boundary/session policy
              -> bucket/access-point policy -> endpoint policy
              -> object ownership/ACL edge case -> KMS key policy/grant
```

Capture exact API, bucket/key, endpoint, region, principal ARN, request ID, and CloudTrail data event if enabled. `ListBucket` applies to the bucket ARN; `GetObject` applies to object ARNs. An SSE-KMS object additionally needs appropriate KMS authorization.

## Secure baseline

- Enable Block Public Access and explicitly review any exception.
- Disable ACL-based ownership patterns where practical; use policies.
- Require TLS and approved encryption through policy conditions.
- Enable versioning and a tested recovery/retention design.
- Log management activity; enable data events selectively based on threat/cost needs.
- Use presigned URLs with short expiry and narrow object/action scope.

## Performance diagnosis

Separate client, network, S3, and KMS symptoms. Examine request latency, first-byte latency, status codes, retries, object sizes, concurrency, DNS path, endpoint/NAT capacity, KMS throttling, and application serialization. Use multipart upload for large transfers and abort incomplete multipart uploads through lifecycle policy.

S3 scales request processing horizontally; modern designs generally do not need randomized prefixes for baseline performance. Hot application behavior can still saturate clients, KMS, NAT, downstream processing or per-connection throughput. Parallelize safely and measure end-to-end.

## Logging, auditing, and recovery

CloudTrail management events record bucket-level management actions; S3 data events can record object APIs at additional volume/cost. S3 server access logging provides request records with delivery/best-effort characteristics. Add AWS Config/Access Analyzer/Security Hub controls as appropriate. Protect the log destination from alteration and avoid recursive logging configurations.

Recovery design can combine versioning, Object Lock, cross-account replication, AWS Backup where supported/appropriate, inventory and tested restore. Define whether recovery restores original keys or a clean destination, how integrity is validated, and how applications are cut over.

## Ordered interview scenarios

### Accidental delete

1. Stop destructive writers and preserve CloudTrail evidence.
2. Determine bucket, prefix, time and whether versioning produced delete markers.
3. Inventory affected versions; restore to a controlled prefix/bucket.
4. Validate count, checksum/business integrity and metadata before cutover.
5. Reduce destructive permissions and test recovery regularly.

### Ransomware protection

1. Use versioning plus Object Lock/immutable backup in a separately administered account where requirements justify it.
2. Separate backup and workload roles/keys; restrict retention changes and permanent deletion.
3. Detect unusual encryption/overwrite/delete/API patterns.
4. Revoke compromised sessions, preserve evidence and restore clean versions in isolation.
5. Test mass-restore time against RTO and key/dependency availability.

### Public bucket incident

1. Confirm exposure through Block Public Access, bucket/access-point policy, ACL and Access Analyzer.
2. Contain with the narrowest safe public-access block/policy correction; preserve logs.
3. Identify exposed objects, duration and access evidence; follow incident/legal process.
4. Rotate any exposed credentials/data where relevant and validate intended application access.
5. Add organization guardrails, IaC policy tests and detection.

### Cross-account access

1. Identify exact caller session, object owner, API and ARN.
2. Check caller identity policy and destination bucket/access-point policy.
3. Check SCP/boundary/endpoint policy, Object Ownership and legacy ACLs.
4. For SSE-KMS, authorize the correct key across accounts.
5. Test the real operation and capture request ID/CloudTrail.

### Upload failure

1. Distinguish DNS/connectivity, HTTP status, signature, authorization, size/checksum and application errors.
2. Verify endpoint/Region, clock, method/key, content headers and presigned constraints.
3. Inspect IAM/bucket/endpoint/KMS policy and quota/throttle signals.
4. For multipart, list parts/status and retry only failed parts idempotently.
5. Abort abandoned uploads and add client telemetry.

### KMS AccessDenied

1. Identify caller, object/version, key ARN and Region from request context.
2. Check identity permission plus key policy/grant and key state.
3. Inspect encryption context, endpoint/SCP conditions and cross-account ownership.
4. Correct the narrow permission and retry one object.
5. Add an integration test using the runtime identity.

### Restrict S3 to a VPC endpoint

1. Add the gateway endpoint to the correct route tables and apply an endpoint policy.
2. Use bucket-policy source-endpoint conditions with explicit exceptions for required trusted paths.
3. Retain identity least privilege and SSE-KMS authorization.
4. Test from allowed and denied networks plus AWS service workflows.
5. Monitor policy drift; endpoint restriction is not a substitute for identity.

### Lifecycle cost optimization

1. Analyze object size, age, access, retrieval urgency and compliance retention.
2. Model storage, request, monitoring, minimum-duration and retrieval charges.
3. Create narrowly filtered transition/expiration rules including noncurrent versions.
4. Pilot and verify inventory/storage-class movement and restore time.
5. Review periodically as access patterns and prices change.

### Replication not working

1. Confirm source/destination versioning and rule filter/status.
2. Inspect replication status/metrics and source object eligibility.
3. Check replication role, destination bucket policy/ownership and KMS permissions.
4. Verify storage-class/Region/Object Lock requirements and failed-operation reports.
5. Correct policy, test new object, then handle existing failed objects explicitly.

### Application receives 403

1. Capture principal, operation, bucket/key, endpoint, Region and request ID.
2. Evaluate identity, bucket/access-point, Block Public Access, endpoint and organization controls.
3. Check object ownership/ACL legacy cases and required bucket versus object ARN.
4. Check KMS key policy/grant for encrypted objects.
5. Apply a minimal correction and verify through the application path.

### Large-file upload strategy

1. Select multipart part size/concurrency from file size, network and S3 limits in current docs.
2. Upload parts with retry, checksum and persistent upload ID; complete only after validation.
3. Use acceleration/edge/client locality only after measurement and cost review.
4. Monitor throughput, retries, NAT/endpoint and KMS signals.
5. Lifecycle-abort incomplete uploads and make completion idempotent.

## Senior interview answers

### S3 versus EBS versus EFS

S3 is object/API storage with massive scale; EBS is low-latency AZ-scoped block storage for an attached compute workload; EFS is regional managed NFS for shared POSIX access. Choose from access semantics first, then performance, resilience, and cost.

### Recover from accidental deletion

Stop writers, identify scope and time, preserve evidence, verify versioning/Object Lock/replicas/backups, restore specific versions to a controlled location, validate integrity, then cut over. Fix permissions and destructive-operation controls before resuming.

## Memory hook

**S3 = API + policy + versions + lifecycle.** “Private” requires proving every access path, not just checking an ACL.

## Official references

- [S3 data consistency](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html#ConsistencyModel)
- [S3 security best practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)
- [S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
