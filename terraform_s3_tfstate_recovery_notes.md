# Terraform State File Deleted from S3 — Production Recovery Runbook

**Audience:** Senior DevOps / Cloud Engineers  
**Scenario:** The Terraform remote state object stored in an S3 bucket was accidentally deleted. The AWS resources still exist, but Terraform may have lost its record of them.

## 1. Objectives and immediate precautions

The goal is to recover Terraform's resource mapping without recreating or disrupting existing infrastructure.

**Immediately:**
1. Pause Terraform CI/CD pipelines, scheduled jobs, and all manual applies for the affected state.
2. Notify the incident owner and identify the AWS account, region, bucket, exact object key, workspace, Terraform version, and approximate deletion time.
3. Do **not** run `terraform apply`, `terraform destroy`, or bulk imports against production until the current state and recovery approach are understood.
4. Do not upload an empty state file to the production key. Do not manually edit state JSON or change its lineage/serial to bypass safeguards.
5. Treat state and plan files as sensitive: they may contain secrets. Store them only in approved, access-controlled locations.

**Important:** Deleting the state file does not delete the actual AWS resources. However, Terraform may no longer know which resources it manages. Running an apply with an empty or incomplete state can propose duplicate resources or risky changes.

## 2. Recovery decision

**Preferred order:** Check for a recoverable S3 object version or a known-good state backup immediately. If a valid, sufficiently recent snapshot exists, restoring it is usually safer than importing every resource. If no trustworthy snapshot is available, reconstruct state by importing existing resources.

### Check S3 versioning and object history

```bash
aws sts get-caller-identity

aws s3api get-bucket-versioning \
  --bucket company-terraform-state

aws s3api list-object-versions \
  --bucket company-terraform-state \
  --prefix production/network/terraform.tfstate
```

- Inspect both `Versions` and `DeleteMarkers`.
- Select the latest **valid version from before deletion**; verify its timestamp and environment.
- Download the candidate version to a secure location using `get-object` and its exact `--version-id`. Validate the file before restoring it.
- Do not permanently delete versions or delete markers during recovery.
- If versioning was not enabled, or the prior version was permanently deleted, look for approved backups and CI/CD artifacts.

Example: download a particular prior version (replace placeholders with verified values):

```bash
aws s3api get-object \
  --bucket company-terraform-state \
  --key production/network/terraform.tfstate \
  --version-id "VERIFIED_VERSION_ID" \
  recovered.tfstate
```

A basic JSON structure check (not proof that this is the correct state):

```bash
jq -e '.version and .serial and .lineage and (.resources | type == "array")' \
  recovered.tfstate
```

Only restore a verified snapshot to the exact original key after considering any infrastructure changes made after that snapshot. Keep the original version history. If there is any uncertainty, involve the state/backend owner before writing to the production key.

## 3. Reconstruct state with Terraform import

Use this path when a trustworthy, sufficiently current state snapshot cannot be recovered. Import associates an **existing** resource with a Terraform resource address; it does not create that resource. Import does not automatically recreate all configuration details.

### Step A — Inventory and map resources

Collect resource IDs from AWS APIs, tags, CloudTrail, deployment records, and configuration. Create a mapping before importing:

| Existing AWS resource | Terraform address | Example ID |
|---|---|---|
| VPC | `aws_vpc.main` | `vpc-0123456789abcdef0` |
| Subnet | `aws_subnet.private_a` | `subnet-0123456789abcdef0` |
| Security group | `aws_security_group.app` | `sg-0123456789abcdef0` |
| EC2 instance | `aws_instance.app` | `i-0123456789abcdef0` |

These IDs are examples only. Confirm the exact resource addresses in the configuration, including module paths, `for_each` keys, `count` indexes, and provider aliases. Ensure the AWS account, region, workspace, and provider configuration are correct.

### Step B — Import one resource at a time

For example, import an existing VPC:

```bash
terraform import \
  aws_vpc.main \
  vpc-0123456789abcdef0
```

For a resource inside a module, the address may look like:

```bash
terraform import \
  'module.network.aws_vpc.main' \
  vpc-0123456789abcdef0
```

Use the exact address and import ID format documented for each resource type. Do not assume all resources use the same ID format. Keep a record of every successful import and any errors. Avoid importing the same real resource to multiple Terraform addresses.

Alternatively, with Terraform versions supporting declarative import blocks:

```hcl
import {
  to = aws_vpc.main
  id = "vpc-0123456789abcdef0"
}
```

Review the import plan and follow your team's approved procedure to complete imports. For a large estate, plan the work by dependency and criticality. Some resource types have special import behavior or require multiple related resources to be imported separately.

### Step C — Check artifacts in parallel

Search secure CI/CD artifacts, approved backups, operator workstations, and files such as `terraform.tfstate.backup`. Compare any recovered snapshot against the inventory and reconstructed state before deciding whether it is safe to use. Do not blindly overwrite or merge states.

## 4. Validate before allowing any apply

First, ensure the correct backend and workspace are selected. Then inspect the imported/recovered state:

```bash
terraform version
terraform workspace show
terraform state list
```

Review whether expected resource addresses are present. Compare critical resources with the real AWS environment. Import alone does not guarantee that the configuration matches existing settings.

Generate and inspect a plan:

```bash
terraform plan -out=recovery-check.tfplan
terraform show recovery-check.tfplan
```

**Stop and investigate** if the plan proposes unexpected resource creation, deletion, or replacement—especially for production VPCs, EKS clusters, RDS databases, IAM policies, or load balancers. Refresh results and plan output must be reviewed; do not apply simply to make the plan “go green.” Protect the plan file because it can contain sensitive data.

Once the state, configuration, and actual infrastructure are reconciled, get the required peer/incident-owner approval before any apply. Save a secure backup of the validated state according to company policy and document the recovery.

## 5. Prevent recurrence

- **S3 Versioning:** Enable it and monitor that it stays enabled. Versioning cannot recover versions permanently deleted before protection was in place.
- **Least privilege:** Restrict state-object writes/deletes and especially `s3:DeleteObjectVersion` to tightly controlled roles.
- **Locking:** Use S3 native state locking (`use_lockfile = true`) where supported (Terraform 1.10+). Locking helps coordinate concurrent Terraform operations; it does not protect against every deletion.
- **Audit and alerting:** Use CloudTrail to monitor state-object deletion, version deletion, bucket-versioning changes, and suspicious IAM changes. Configure alerts for relevant events.
- **Backups and DR:** Keep encrypted, access-controlled backups with suitable retention; consider a separate account and periodically test recovery.
- **Operational controls:** Require reviewed pipeline changes, restrict manual production access, document the backend/workspace map, and test the runbook in a non-production environment.

Example S3 backend configuration (adapt to your environment; make backend changes only through an approved process):

```hcl
terraform {
  backend "s3" {
    bucket       = "company-terraform-state"
    key          = "production/network/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

## Incident checklist

- [ ] Terraform pipelines and manual applies frozen
- [ ] Correct account, bucket, key, region, workspace confirmed
- [ ] S3 versions/delete markers and approved backups checked
- [ ] Existing AWS resource inventory and address mapping prepared
- [ ] State restored or existing resources imported carefully
- [ ] `terraform state list` and resource inventory verified
- [ ] Plan reviewed; no unexplained creates/deletes/replacements
- [ ] Peer approval obtained before any production apply
- [ ] Root cause investigated; versioning, IAM, audit alerts, backups tested

**Official references**
- Terraform S3 backend: https://developer.hashicorp.com/terraform/language/backend/s3
- Terraform state: https://developer.hashicorp.com/terraform/language/state
- AWS S3 versioning: https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html
