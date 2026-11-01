# Terraform State File Locking — Production Notes

<img width="1862" height="418" alt="image" src="https://github.com/user-attachments/assets/376693b4-0607-4c13-b210-c07ec16b7959" />

<img width="1003" height="902" alt="image" src="https://github.com/user-attachments/assets/8aa5bc6c-efbf-43eb-aecd-555c28b4cc3f" />



## 1. What is Terraform state locking?

Terraform state (`terraform.tfstate`) records the infrastructure resources Terraform manages. **State locking** prevents multiple Terraform operations from concurrently changing the **same state**, helping avoid conflicting updates and state corruption. Locking is supported by certain backends, including the Amazon S3 backend.

### Real-world scenario: two DevOps engineers

1. **Engineer 1** runs `terraform apply` against the production state and successfully **acquires the lock**.
2. **Engineer 2** runs `terraform apply` against the **same backend state key** while Engineer 1 is still working.
3. Terraform cannot grant Engineer 2 the same lock. By default, Engineer 2 typically sees **`Error acquiring the state lock`**; with a lock timeout configured, Terraform retries until the lock is released or the timeout expires.
4. After Engineer 1 finishes, Terraform **releases the lock**. Engineer 2 can then retry (or proceed automatically if still waiting), with Terraform refreshing/planning against the current state.

> The first engineer to *successfully acquire the backend lock* wins—not necessarily the one who typed the command first. Locking coordinates Terraform state operations; it does not stop manual changes to AWS resources outside Terraform.

## 2. Traditional approach: S3 remote state + DynamoDB lock

Store the state in **Amazon S3** and use a **DynamoDB table** for lock coordination. Create the bucket and a DynamoDB table named `terraform-state-lock` with a **String partition key `LockID`** before initializing the backend.

**`backend.tf` (legacy approach)**

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "ap-south-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

**How it works:** Terraform uses DynamoDB to acquire/release the lock, while the state itself is stored in S3. **DynamoDB-based state locking is deprecated** for the S3 backend; prefer native S3 locking for new setups.

## 3. Modern approach: native S3 lockfile (Terraform 1.10+)

The S3 backend supports native locking by adding **one parameter**: `use_lockfile = true`. No DynamoDB table is needed.

**`backend.tf` (recommended)**

```hcl
terraform {
  backend "s3" {
    bucket       = "my-terraform-state"
    key          = "production/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

**Behind the scenes:** Terraform creates a lock object alongside the state, e.g. `production/terraform.tfstate.tflock`, to coordinate access. A competing Terraform process cannot acquire the same lock until it is released. Terraform removes the lock object when the operation ends normally.

**Initialize and apply:**

```bash
terraform init
terraform plan
terraform apply
```

For an **existing** backend whose configuration changes, coordinate the migration and use `terraform init -reconfigure` where appropriate. During a transition, the S3 backend can be configured with both DynamoDB and S3 locking to support migration; ensure all engineers and CI/CD jobs follow a compatible locking strategy.

## 4. Lock error, timeout, and safe recovery

Illustrative error from Engineer 2:

```text
$ terraform apply
Error: Error acquiring the state lock
Lock Info:
  Operation: OperationTypeApply
  Who:       engineer1@workstation
```

To wait up to **5 minutes** for the lock:

```bash
terraform apply -lock-timeout=5m
```

If the lock remains held beyond the timeout, Terraform returns an error. **Do not routinely use `-lock=false`**, because it bypasses locking safeguards. Use `terraform force-unlock <LOCK_ID>` only after confirming the original operation is no longer running and the lock is genuinely stale; force-unlocking an active operation is dangerous.

## 5. Quick comparison

| Feature | S3 + DynamoDB (legacy) | S3 native locking |
|---|---|---|
| State storage | S3 | S3 |
| Lock storage | DynamoDB item | S3 `.tflock` object |
| Backend setting | `dynamodb_table` | `use_lockfile = true` |
| Separate DynamoDB table | Yes | No |
| Recommendation | Deprecated | Preferred for new setups |

## 6. Production best practices

- Enable **S3 bucket versioning** for state recovery; enable encryption and restrict bucket access.
- Use **least-privilege IAM**. Native locking needs S3 permissions to read/write the state and **GetObject, PutObject, DeleteObject** on the `.tflock` object; do not broadly expose state or locks.
- Store separate environment states under separate keys (e.g. `dev/terraform.tfstate` and `production/terraform.tfstate`).
- Run production applies through a controlled CI/CD workflow and keep the Terraform version/backend configuration consistent across operators.
- Never commit sensitive state to Git; state may contain secrets even when Terraform outputs mark values sensitive.

**Interview takeaway:** *Terraform state locking ensures that only one operation can hold the lock for a shared state at a time. Historically, we used S3 plus DynamoDB; with Terraform 1.10+, native S3 locking can be enabled using `use_lockfile = true`.*

**Official references:**
- [Terraform S3 backend documentation](https://developer.hashicorp.com/terraform/language/backend/s3)
- [Terraform state locking documentation](https://developer.hashicorp.com/terraform/language/state/locking)
