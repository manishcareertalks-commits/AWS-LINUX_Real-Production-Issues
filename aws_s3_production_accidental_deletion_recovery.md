# AWS S3 Production Incident Runbook: Accidental File Deletion & Recovery

<img width="934" height="910" alt="image" src="https://github.com/user-attachments/assets/194ef914-81a6-43c5-b558-2792997d1810" />

## 1. Incident Overview

### Scenario

A developer accidentally deletes an important object from a production
Amazon S3 bucket.

Example:

``` text
s3://production-data/invoices/invoice-1001.pdf
```

The application subsequently returns:

``` text
HTTP 404 Not Found
```

or reports that the file does not exist.

The immediate questions are:

1.  Was S3 Versioning enabled before the deletion?
2.  Was the object actually deleted, or is a Delete Marker hiding an
    older version?
3.  Was a specific object version permanently deleted?
4.  Do we have an independent backup or replica?
5.  Who performed the delete operation and why?
6.  How do we prevent a recurrence?

------------------------------------------------------------------------

# 2. Production Architecture

A typical production flow may look like:

``` text
                         ┌───────────────────┐
                         │       Users       │
                         └─────────┬─────────┘
                                   │
                                   ▼
                         ┌───────────────────┐
                         │  Application/API  │
                         └─────────┬─────────┘
                                   │
                                   ▼
                         ┌───────────────────┐
                         │    S3 Bucket      │
                         │ production-data   │
                         └─────────┬─────────┘
                                   │
                ┌──────────────────┼──────────────────┐
                ▼                  ▼                  ▼
          Versioning           Logging/Audit      Replication
                │                  │                  │
                ▼                  ▼                  ▼
        Previous versions      CloudTrail       DR bucket/Region
```

For business-critical data, S3 should be treated as part of a broader
data-protection strategy rather than relying on a single control.

------------------------------------------------------------------------

# 3. What Happens When an S3 Object Is Deleted?

## Without Versioning

If versioning is not enabled, a normal delete can remove the current
object.

Depending on the situation and other controls, recovery may require an
independent backup, replica, or other recovery mechanism.

Therefore:

> If you need protection from accidental deletion, configure the
> protection before the incident.

------------------------------------------------------------------------

## With Versioning Enabled

When Versioning is enabled, S3 can retain multiple versions of an
object.

For example:

``` text
invoice-1001.pdf
       │
       ├── Version A  ← old version
       ├── Version B  ← latest version
       └── Delete Marker
```

For a normal delete request without specifying a version ID, S3 adds a
**Delete Marker** instead of permanently deleting the current object
version.

The object can therefore appear deleted to normal GET/list operations
while an older version remains stored.

### Important distinction

A Delete Marker is not the same as permanently deleting an object
version.

This distinction is critical during production recovery.

------------------------------------------------------------------------

# 4. Example Incident

Assume the bucket contains:

``` text
production-data/
└── invoices/
    └── invoice-1001.pdf
```

Before the incident:

``` text
Version ID: v2
Status: Current
```

A developer executes:

``` bash
aws s3 rm s3://production-data/invoices/invoice-1001.pdf
```

With Versioning enabled, S3 creates a Delete Marker.

Conceptually:

``` text
Before:

invoice-1001.pdf
       │
       └── v2 (current)


After accidental delete:

invoice-1001.pdf
       │
       ├── Delete Marker ← current
       └── v2            ← previous version still retained
```

A normal application request can now behave as though the object does
not exist.

------------------------------------------------------------------------

# 5. First Response: Production Incident Procedure

Do not immediately start deleting or modifying objects.

Use a controlled incident-response process.

## Step 1 --- Stop Further Destructive Operations

If the deletion was caused by a script, deployment, compromised
credential, or incorrect automation:

-   Stop the job.
-   Disable the affected automation if necessary.
-   Revoke or rotate compromised credentials.
-   Prevent additional deletes.
-   Preserve evidence.

The goal is to prevent the incident from becoming larger.

------------------------------------------------------------------------

## Step 2 --- Identify the Exact Object

Record:

``` text
Bucket:
Object key:
Approximate deletion time:
Application/service:
Environment:
Business owner:
```

Example:

``` text
Bucket: production-data
Key: invoices/invoice-1001.pdf
Environment: production
```

Avoid making broad changes to the bucket when only one object is
affected.

------------------------------------------------------------------------

# 6. Check Whether Versioning Is Enabled

Using AWS CLI:

``` bash
aws s3api get-bucket-versioning \
  --bucket production-data
```

Example response:

``` json
{
  "Status": "Enabled"
}
```

If Versioning is enabled, continue investigating object versions.

If it is not enabled, move to the independent recovery options described
later in this document.

------------------------------------------------------------------------

# 7. Identify Object Versions

Use:

``` bash
aws s3api list-object-versions \
  --bucket production-data \
  --prefix invoices/invoice-1001.pdf
```

You may see something conceptually similar to:

``` text
Versions:
  VersionId: v2
  IsLatest: false
  Key: invoices/invoice-1001.pdf

DeleteMarkers:
  VersionId: d1
  IsLatest: true
  Key: invoices/invoice-1001.pdf
```

This is the key recovery clue.

The Delete Marker is currently the latest version, while the previous
object version still exists.

------------------------------------------------------------------------

# 8. Recovery Method: Remove the Delete Marker

For a normal versioned delete, removing the Delete Marker can make the
previous object version current again.

First, identify the Delete Marker version ID.

Then perform a version-specific delete against the Delete Marker:

``` bash
aws s3api delete-object \
  --bucket production-data \
  --key invoices/invoice-1001.pdf \
  --version-id <DELETE-MARKER-VERSION-ID>
```

Afterward, verify the object.

``` bash
aws s3api head-object \
  --bucket production-data \
  --key invoices/invoice-1001.pdf
```

If the previous version is now current, the application can access the
object again.

### Important

This is not the same as deleting the object permanently.

You are specifically removing the Delete Marker that was hiding the
previous version.

------------------------------------------------------------------------

# 9. Alternative Recovery: Copy a Specific Version

Sometimes you need to restore a particular historical version rather
than simply removing the Delete Marker.

First identify the required version:

``` bash
aws s3api list-object-versions \
  --bucket production-data \
  --prefix invoices/invoice-1001.pdf
```

Then retrieve or copy the desired version as required by the recovery
procedure.

For example, you can download a specific version:

``` bash
aws s3api get-object \
  --bucket production-data \
  --key invoices/invoice-1001.pdf \
  --version-id <VERSION-ID> \
  restored-invoice-1001.pdf
```

You can then validate the file and restore it through a controlled
process.

This approach is useful when:

-   Multiple versions exist.
-   The latest previous version is not the correct one.
-   The object was overwritten before deletion.
-   You need to compare historical versions.

------------------------------------------------------------------------

# 10. Critical Warning: Version IDs Matter

Do not confuse these operations:

``` text
Delete object normally
        ↓
Creates Delete Marker when Versioning is enabled


Delete a specific version ID
        ↓
Permanently removes that particular version
```

If someone permanently deletes the required version using its Version
ID, Versioning alone cannot magically recreate it.

This is one reason why Versioning should not be treated as a complete
backup strategy.

------------------------------------------------------------------------

# 11. What If Versioning Was NOT Enabled?

If the object was deleted and no recoverable version exists, investigate
independent recovery mechanisms.

Potential sources include:

### 1. S3 Replication

If the object was replicated to another bucket/Region, determine whether
a usable copy exists there.

Replication is useful for resilience and disaster recovery, but its
exact behavior depends on the replication configuration and event that
caused the deletion.

### 2. AWS Backup

If the S3 bucket/data was included in an appropriate AWS Backup
configuration, investigate the available recovery point.

### 3. Application Backup

Some applications maintain independent copies of important documents.

### 4. Data Pipeline / Source System

The original file may still exist in:

-   Database
-   File server
-   Data lake
-   Upload service
-   User system
-   Another S3 bucket

### 5. Business Owner Recovery

For business-critical files, the owner may have the original source
document.

------------------------------------------------------------------------

# 12. S3 Versioning Is NOT a Complete Backup Strategy

This is one of the most important production lessons.

Versioning protects against scenarios such as:

-   Accidental deletion
-   Accidental overwrite
-   Application writing an incorrect version

But Versioning has limitations.

For example:

``` text
Application mistake
       ↓
Bad file uploaded
       ↓
Bad version becomes current
       ↓
Versioning retains history
```

This is helpful.

But if an attacker or privileged user intentionally performs
version-specific permanent deletion, the retained version can also be
removed.

Therefore:

``` text
Versioning
     +
Independent Backup
     +
Access Control
     +
Monitoring
     +
Recovery Testing
```

provides a much stronger production posture.

------------------------------------------------------------------------

# 13. Protecting Important S3 Data

## A. Enable Versioning

For production buckets where recovery from accidental overwrites/deletes
matters:

``` text
S3 Bucket
   ↓
Versioning = Enabled
```

Versioning should be enabled before the incident occurs.

It is not a time machine for objects that were permanently deleted
before versioning protection existed.

------------------------------------------------------------------------

## B. Use Least-Privilege IAM

Do not give developers unrestricted production S3 permissions.

Avoid broad permissions such as:

``` text
s3:*
```

when they are not required.

Prefer narrowly scoped permissions.

For example:

``` text
Developer
   ↓
Read production data
   ↓
No permanent-delete permission
```

Separate operational roles from destructive administrative roles.

------------------------------------------------------------------------

# 14. Protect Against Permanent Version Deletion

A common production mistake is protecting the current object while
allowing users or compromised credentials to permanently delete object
versions.

Review permissions involving:

``` text
s3:DeleteObject
s3:DeleteObjectVersion
```

Pay particular attention to:

``` text
s3:DeleteObjectVersion
```

because version-specific deletion can permanently remove a retained
version.

Use IAM policies, permission boundaries, SCPs, and controlled
administrative roles where appropriate.

------------------------------------------------------------------------

# 15. Object Lock

For highly important data, consider **S3 Object Lock** where the
workload and compliance requirements justify it.

Object Lock can provide WORM-style protection.

Conceptually:

``` text
Application
     │
     ▼
S3 Object
     │
     ▼
Retention / Legal Hold
     │
     ▼
Object protected from deletion
```

Object Lock is especially relevant for:

-   Compliance records
-   Financial records
-   Audit data
-   Critical logs
-   Regulatory retention

Choose the retention model carefully because protection can
intentionally make deletion difficult.

------------------------------------------------------------------------

# 16. MFA Delete

For certain high-risk versioned bucket operations, MFA Delete can add an
additional authentication requirement.

It is important to understand exactly what it protects and the
operational implications before adopting it.

Do not treat MFA Delete as a replacement for IAM controls, backups, or
monitoring.

------------------------------------------------------------------------

# 17. S3 Lifecycle Policies

Versioning can cause multiple versions to accumulate.

Example:

``` text
Current version
     +
Old version
     +
Older version
     +
Delete marker
     +
More historical versions
```

Without lifecycle management, storage costs can increase significantly.

Consider lifecycle rules for:

-   Noncurrent versions
-   Expired delete markers
-   Incomplete multipart uploads
-   Transitioning suitable data to lower-cost storage classes

However, be careful with aggressive expiration rules.

A lifecycle rule that deletes old versions too quickly can reduce your
recovery window.

------------------------------------------------------------------------

# 18. Cross-Region Replication

For disaster recovery requirements, consider S3 replication to another
bucket/Region.

Example:

``` text
              Primary Region
          ┌──────────────────┐
          │ Production S3    │
          │ Bucket           │
          └────────┬─────────┘
                   │
              Replication
                   │
                   ▼
          ┌──────────────────┐
          │ DR S3 Bucket     │
          │ Other Region     │
          └──────────────────┘
```

Replication can improve resilience against regional or operational
failures.

However, replication is not automatically equivalent to a backup.

Design and test the replication behavior for deletes, overwrites,
versioning, ownership, permissions, and the specific recovery scenario.

------------------------------------------------------------------------

# 19. Audit: Who Deleted the File?

After recovery, determine who or what performed the deletion.

Use AWS audit/logging capabilities such as **AWS CloudTrail** to
investigate API activity.

Look for events related to:

``` text
DeleteObject
DeleteObjects
DeleteObjectVersion
```

Useful investigation details include:

``` text
Who?
When?
Which bucket?
Which object?
Which AWS account?
Which IAM role?
Which source IP?
Which application/automation?
```

This changes the incident from:

> "Someone deleted a file."

to:

> "Role X performed a DeleteObject operation on object Y at time Z."

That is much more actionable.

------------------------------------------------------------------------

# 20. Monitoring and Alerting

Production S3 environments should have monitoring around sensitive
operations.

Consider alerting on:

``` text
Unexpected object deletion
Large-scale deletion
Version deletion
Bucket policy changes
Public access changes
IAM permission changes
Replication failures
```

A simplified detection flow:

``` text
S3 Activity
     │
     ▼
CloudTrail / Security Monitoring
     │
     ▼
Detection Rule
     │
     ▼
Alert
     │
     ▼
SRE / Security Team
```

The goal is to detect destructive activity quickly.

------------------------------------------------------------------------

# 21. Incident Response Runbook

Use the following checklist during a real incident.

## Phase 1 --- Detect

``` text
[ ] Application reports missing object
[ ] Confirm affected bucket
[ ] Confirm affected object/key
[ ] Determine incident start time
```

## Phase 2 --- Contain

``` text
[ ] Stop destructive automation
[ ] Check for ongoing mass deletion
[ ] Disable compromised credentials if required
[ ] Preserve logs/evidence
```

## Phase 3 --- Investigate

``` text
[ ] Check S3 Versioning
[ ] List object versions
[ ] Check for Delete Marker
[ ] Identify required Version ID
[ ] Check CloudTrail
[ ] Identify actor/role
```

## Phase 4 --- Recover

``` text
[ ] Remove Delete Marker if appropriate
[ ] Restore/copy required version
[ ] Validate object integrity
[ ] Verify application access
[ ] Monitor for recurrence
```

## Phase 5 --- Prevent

``` text
[ ] Review IAM permissions
[ ] Review DeleteObjectVersion permissions
[ ] Review Versioning
[ ] Review Object Lock requirements
[ ] Review backup strategy
[ ] Review replication
[ ] Review lifecycle policies
[ ] Add monitoring/alerts
[ ] Test recovery procedure
```

------------------------------------------------------------------------

# 22. Example Production Recovery Flow

``` text
             INCIDENT
                 │
                 ▼
       File returns HTTP 404
                 │
                 ▼
       Identify S3 object/key
                 │
                 ▼
       Check Versioning status
                 │
          ┌──────┴──────┐
          │             │
       Enabled        Disabled
          │             │
          ▼             ▼
 List object versions   Check
          │             independent
          │             backups/DR
          ▼
   Delete Marker?
      │        │
     Yes       No
      │        │
      ▼        ▼
Remove marker  Find appropriate
      │        version/backup
      └────┬───────┘
           ▼
      Validate object
           │
           ▼
    Application works
           │
           ▼
     Root-cause analysis
           │
           ▼
   Prevent recurrence
```

------------------------------------------------------------------------

# 23. Common Production Mistakes

## Mistake 1 --- "S3 is durable, so deletion is impossible"

Incorrect.

Durability does not mean protection from an authorized or accidental
delete operation.

------------------------------------------------------------------------

## Mistake 2 --- "Versioning is the same as backup"

Incorrect.

Versioning provides historical object versions but should not
automatically be considered an independent backup strategy.

------------------------------------------------------------------------

## Mistake 3 --- Giving developers `s3:*`

This increases the blast radius of:

-   Human mistakes
-   Compromised credentials
-   Malicious actions
-   Incorrect automation

------------------------------------------------------------------------

## Mistake 4 --- Ignoring DeleteObjectVersion

A team may enable Versioning but still allow privileged users to
permanently delete historical versions.

Review this permission carefully.

------------------------------------------------------------------------

## Mistake 5 --- Never testing recovery

Having Versioning enabled is not enough.

You should periodically verify:

``` text
Can we find the correct version?
Can we restore it?
Can the application read it?
Can we recover within our RTO?
```

------------------------------------------------------------------------

# 24. RPO and RTO Considerations

Production recovery should be aligned with business requirements.

## RPO --- Recovery Point Objective

How much data can the business afford to lose?

Example:

``` text
RPO = 15 minutes
```

This affects backup and replication design.

## RTO --- Recovery Time Objective

How quickly must the service/data be restored?

Example:

``` text
RTO = 30 minutes
```

This affects how quickly your team must be able to identify and restore
data.

------------------------------------------------------------------------

# 25. Production Design Recommendation

For business-critical S3 data, a mature design may look like:

``` text
                         Production Application
                                  │
                                  ▼
                         ┌─────────────────┐
                         │   S3 Bucket     │
                         │   Versioning    │
                         └────────┬────────┘
                                  │
                ┌─────────────────┼─────────────────┐
                │                 │                 │
                ▼                 ▼                 ▼
          IAM Controls        CloudTrail        Monitoring
                │                 │                 │
                │                 ▼                 ▼
                │             Audit Trail         Alerts
                │
                ▼
        Restricted Delete
                │
                ▼
          Object Lock
        where appropriate
                │
                └─────────────────┐
                                  ▼
                         Independent Backup
                                  │
                                  ▼
                         DR / Replication
```

The exact architecture should be based on data criticality, compliance
requirements, RPO/RTO, cost, and operational needs.

------------------------------------------------------------------------

# 26. Interview Question

### Question

**"A developer accidentally deleted a production file from S3. How would
you recover it?"**

### Strong Answer

> "First, I would contain the incident and make sure no further
> destructive operations are occurring. Then I would verify whether S3
> Versioning was enabled before the deletion. If it was enabled, I would
> list the object's versions and check whether the latest entry is a
> Delete Marker. If the required previous version exists, I can restore
> the object by removing the Delete Marker or by restoring/copying the
> required version. I would then validate the application and
> investigate CloudTrail to identify what role or process performed the
> deletion. Finally, I would review IAM permissions, version-deletion
> permissions, backups, lifecycle policies, replication, monitoring, and
> recovery testing so the same incident does not happen again."

This answer demonstrates:

``` text
AWS Knowledge
+
Incident Response
+
Troubleshooting
+
Security
+
Disaster Recovery
+
Production Thinking
```

------------------------------------------------------------------------

# 27. Key Takeaways

### Remember these five points:

**1. Enable Versioning before you need it.**

**2. A normal delete with Versioning can create a Delete Marker rather
than immediately destroying the previous version.**

**3. A specific version can be permanently deleted, so Versioning alone
is not a complete backup strategy.**

**4. Use least-privilege IAM and carefully control destructive
permissions.**

**5. Test your recovery process before a real production incident
happens.**

------------------------------------------------------------------------

# 28. One-Line Production Rule

> **The best time to prepare for an accidental S3 deletion is before the
> deletion happens.**

``` text
S3 Versioning
      +
Backup / DR
      +
Least-Privilege IAM
      +
Monitoring
      +
Recovery Testing
      =
Stronger Production Data Protection
```

------------------------------------------------------------------------

## AWS CLI Quick Reference

### Check Versioning

``` bash
aws s3api get-bucket-versioning \
  --bucket production-data
```

### List object versions

``` bash
aws s3api list-object-versions \
  --bucket production-data \
  --prefix invoices/invoice-1001.pdf
```

### Download a specific version

``` bash
aws s3api get-object \
  --bucket production-data \
  --key invoices/invoice-1001.pdf \
  --version-id <VERSION-ID> \
  restored-invoice-1001.pdf
```

### Remove a Delete Marker

``` bash
aws s3api delete-object \
  --bucket production-data \
  --key invoices/invoice-1001.pdf \
  --version-id <DELETE-MARKER-VERSION-ID>
```

### Verify the restored/current object

``` bash
aws s3api head-object \
  --bucket production-data \
  --key invoices/invoice-1001.pdf
```

------------------------------------------------------------------------

# Final Message for Your Instagram Reel

**Production incident:** Developer accidentally deleted an S3 file.

**Root protection:** S3 Versioning.

**Recovery:** Identify the Delete Marker and recover the required
previous version.

**Production lesson:** Versioning is protection, not a complete backup
strategy.

**Senior-level thinking:** Add IAM controls, audit logging, monitoring,
backup/DR, appropriate retention controls, and regularly test recovery.

------------------------------------------------------------------------

## Disclaimer

This document is intended for educational and production-design
discussion. AWS behavior and available features can vary by
configuration and service updates. Always test recovery procedures in a
non-production environment and validate the current AWS documentation
before applying destructive or recovery commands to production data.
