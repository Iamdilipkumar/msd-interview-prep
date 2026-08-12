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
