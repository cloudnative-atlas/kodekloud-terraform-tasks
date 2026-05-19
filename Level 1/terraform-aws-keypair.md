# Terraform: AWS Key Pair (datacenter-kp) on LocalStack

A Terraform configuration that provisions an AWS EC2 key pair (against a LocalStack backend), generates the matching RSA private key locally, and writes the `.pem` file to a specific path on disk. Part of the Nautilus DevOps team's incremental AWS migration.

---

## Task

The Nautilus DevOps team is strategizing the migration of a portion of their infrastructure to the AWS cloud. Recognizing the scale of this undertaking, they have opted to approach the migration in incremental steps rather than as a single massive transition. To achieve this, they have segmented large tasks into smaller, more manageable units. This granular approach enables the team to execute the migration in gradual phases, ensuring smoother implementation and minimizing disruption to ongoing operations. By breaking down the migration into smaller tasks, the Nautilus DevOps team can systematically progress through each stage, allowing for better control, risk mitigation, and optimization of resources throughout the migration process.

For this task, create a key pair using Terraform with the following requirements:

* Name of the `key pair` should be `datacenter-kp`.
* Key pair `type` must be `rsa`.
* The private key file should be saved under `/home/bob/datacenter-kp.pem`.

The Terraform working directory is `/home/bob/terraform`. Create the `main.tf` file (do not create a different `.tf` file) to accomplish this task.

**Note:** Right-click under the `EXPLORER` section in `VS Code` and select `Open in Integrated Terminal` to launch the terminal.

---

## Important Lab Context

This is **not real AWS**. The lab uses **LocalStack** as a local AWS emulator running at `http://aws:4566`. A pre-existing `/home/bob/terraform/provider.tf` already declares and configures the AWS provider with all the LocalStack endpoint overrides:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "5.91.0"
    }
  }
}

provider "aws" {
  region                      = "us-east-1"
  skip_credentials_validation = true
  skip_requesting_account_id  = true
  s3_use_path_style           = true

  endpoints {
    ec2 = "http://aws:4566"
    # ...all other services pointed at LocalStack
  }
}
```

This has two practical consequences for `main.tf`:

1. **Do not declare the AWS provider again.** The version pin (`5.91.0`), the region, the LocalStack endpoint overrides, and the credential-validation skips are all configured in `provider.tf`. Duplicating any of this in `main.tf` causes "duplicate configuration" errors.
2. **Do not modify `provider.tf`.** The task instruction "do not create a different .tf file" implies the existing file structure is authoritative. The LocalStack endpoints are specifically tuned for this lab environment.

The `tls` and `local` providers used in `main.tf` do not interact with AWS, so they are unaffected by the LocalStack setup. Terraform will auto-discover them during `terraform init`.

---

## Key Design Choices

The `aws_key_pair` resource imports an existing public key into AWS; it does not generate the key material itself. To satisfy all three requirements (named key pair in AWS, RSA type, local `.pem` file), the configuration uses three resources working together:

1. **`tls_private_key`** generates an RSA key pair in memory during Terraform apply.
2. **`aws_key_pair`** imports the public half into LocalStack's EC2 emulation under the name `datacenter-kp`. Because the supplied public key is RSA, the recorded key type is `rsa` automatically.
3. **`local_file`** writes the private half to `/home/bob/datacenter-kp.pem` with restrictive permissions so it is usable for SSH without triggering "permissions too open" warnings.

Terraform's dependency graph resolves the ordering: `local_file` and `aws_key_pair` both depend on `tls_private_key`, so the key is generated first and consumed by the other two resources in parallel.

---

## Solution

`/home/bob/terraform/main.tf`:

```hcl
# Generate an RSA key pair in memory during apply
resource "tls_private_key" "datacenter_kp" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

# Import the public key into AWS (LocalStack) as a named key pair
resource "aws_key_pair" "datacenter_kp" {
  key_name   = "datacenter-kp"
  public_key = tls_private_key.datacenter_kp.public_key_openssh
}

# Persist the private key to disk so it can be used for SSH
resource "local_file" "private_key_pem" {
  content         = tls_private_key.datacenter_kp.private_key_pem
  filename        = "/home/bob/datacenter-kp.pem"
  file_permission = "0400"
}
```

Notice what is **not** in this file:

- No `terraform { required_providers }` block. The AWS provider is already pinned in `provider.tf`. The `tls` and `local` providers will be auto-discovered.
- No `provider "aws"` block. Already configured in `provider.tf` with all the LocalStack endpoints.

Adding either of those here would create duplicate-configuration errors.

---

## Running the Configuration

### Step 1: `terraform init`

```bash
cd /home/bob/terraform
terraform init
```

Actual output:

```
Initializing the backend...
Initializing provider plugins...
- Finding latest version of hashicorp/local...
- Finding latest version of hashicorp/tls...
- Finding hashicorp/aws versions matching "5.91.0"...
- Installing hashicorp/local v2.9.0...
- Installed hashicorp/local v2.9.0 (signed by HashiCorp)
- Installing hashicorp/tls v4.3.0...
- Installed hashicorp/tls v4.3.0 (signed by HashiCorp)
- Installing hashicorp/aws v5.91.0...
- Installed hashicorp/aws v5.91.0 (signed by HashiCorp)

Terraform has been successfully initialized!
```

Note that AWS is pinned to `5.91.0` (from `provider.tf`), while `tls` and `local` resolve to their latest compatible versions (`4.3.0` and `2.9.0` respectively).

### Step 2: `terraform apply`

```bash
terraform apply -auto-approve
```

Actual plan and apply output:

```
Terraform will perform the following actions:

  # aws_key_pair.datacenter_kp will be created
  + resource "aws_key_pair" "datacenter_kp" {
      + key_name        = "datacenter-kp"
      + key_type        = (known after apply)
      + public_key      = (known after apply)
      # ...
    }

  # local_file.private_key_pem will be created
  + resource "local_file" "private_key_pem" {
      + content              = (sensitive value)
      + file_permission      = "0400"
      + filename             = "/home/bob/datacenter-kp.pem"
      # ...
    }

  # tls_private_key.datacenter_kp will be created
  + resource "tls_private_key" "datacenter_kp" {
      + algorithm                     = "RSA"
      + rsa_bits                      = 2048
      # ...
    }

Plan: 3 to add, 0 to change, 0 to destroy.

tls_private_key.datacenter_kp: Creating...
tls_private_key.datacenter_kp: Creation complete after 0s [id=1bc3a79e23b7349463bcebcc3813fe3c3600b5be]
local_file.private_key_pem: Creating...
local_file.private_key_pem: Creation complete after 0s [id=75b929b8737b487ff8b92ece1be6d26d9e8b5634]
aws_key_pair.datacenter_kp: Creating...
aws_key_pair.datacenter_kp: Creation complete after 1s [id=datacenter-kp]

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
```

Observe the creation order matches the dependency graph: `tls_private_key` first (its output feeds the other two), then `local_file` and `aws_key_pair` in parallel.

---

## Verification

Because the AWS provider is pointed at LocalStack, standard `aws` CLI calls will not see this key pair unless they also target the LocalStack endpoint. In this lab, both forms work because the CLI appears to be pre-wired for LocalStack via environment variables.

### Option 1: Pre-wired CLI

```bash
aws ec2 describe-key-pairs --key-name datacenter-kp
```

### Option 2: Explicit endpoint

```bash
aws --endpoint-url=http://aws:4566 ec2 describe-key-pairs --key-name datacenter-kp
```

Both return identical output:

```json
{
    "KeyPairs": [
        {
            "KeyPairId": "key-a85a609dead882e3b",
            "CreateTime": "2026-05-19T10:58:21.928000Z",
            "KeyName": "datacenter-kp",
            "KeyFingerprint": "29:db:91:5a:84:18:b7:37:6a:22:a8:c5:3e:03:e9:b8"
        }
    ]
}
```

**A LocalStack quirk to be aware of:** the `KeyType` field is omitted from this response, even though real AWS would include it. The key is still RSA (the supplied public key was RSA-formatted, generated by `tls_private_key` with `algorithm = "RSA"`), but LocalStack's EC2 emulation does not surface that field in `describe-key-pairs`. Do not interpret its absence as a problem with your configuration.

If you want to inspect the local `.pem` file:

```bash
ls -la /home/bob/datacenter-kp.pem
head -1 /home/bob/datacenter-kp.pem
```

```
-r-------- 1 bob bob 1675 ... /home/bob/datacenter-kp.pem
-----BEGIN RSA PRIVATE KEY-----
```

The `0400` permissions and the `RSA PRIVATE KEY` PEM header together confirm both the local artifact and the RSA algorithm.

---

## Explanation of the Solution

### 1. RSA Key Generation

```hcl
resource "tls_private_key" "datacenter_kp" {
  algorithm = "RSA"
  rsa_bits  = 2048
}
```

The `algorithm = "RSA"` argument satisfies the task's "type must be `rsa`" requirement at the source: the generated public key is in RSA format, so when imported into the AWS resource the recorded `KeyType` is automatically `rsa`. `rsa_bits = 2048` is the standard size.

The key material is generated during `terraform apply` and stored in Terraform state. The state file therefore contains sensitive material and should not be committed to source control.

### 2. AWS Key Pair Import

```hcl
resource "aws_key_pair" "datacenter_kp" {
  key_name   = "datacenter-kp"
  public_key = tls_private_key.datacenter_kp.public_key_openssh
}
```

This resource imports the public half of the key into LocalStack's EC2 emulation under the exact name required by the task. `aws_key_pair` does not generate keys, it imports an existing public key. The `key_type` field is not set explicitly because it is computed from the public key format. Setting it manually would be redundant and could create a mismatch if the algorithms ever drifted apart.

### 3. Private Key Persistence

```hcl
resource "local_file" "private_key_pem" {
  content         = tls_private_key.datacenter_kp.private_key_pem
  filename        = "/home/bob/datacenter-kp.pem"
  file_permission = "0400"
}
```

The private key is written to disk so it can be used for SSH access. `file_permission = "0400"` sets read-only-for-owner, which is required for OpenSSH to accept the key (it refuses keys with broader permissions). The task does not explicitly require these permissions, but SSH-usable keys are the implicit point of writing the file, so this default is what makes the artifact useful.

---

## Common Pitfalls

| Symptom | Likely Cause | Fix |
|---|---|---|
| `Error: Duplicate provider configuration` during `init` or `plan` | `main.tf` also declares `provider "aws"`, conflicting with `provider.tf` | Remove the `provider "aws"` block from `main.tf`. The pre-existing `provider.tf` is authoritative. |
| `Error: Duplicate required providers configuration` | `main.tf` has its own `terraform { required_providers { aws = ... } }` block that conflicts with `provider.tf` | Remove the AWS entry from `main.tf`'s `required_providers`, or remove the whole block (auto-discovery still works for `tls` and `local`) |
| `KeyType` field missing from `describe-key-pairs` output | LocalStack quirk; real AWS would include it | Not a problem. The key is RSA because `tls_private_key` was set with `algorithm = "RSA"`. Verify via the `-----BEGIN RSA PRIVATE KEY-----` header in the `.pem` file. |
| `aws ec2 describe-key-pairs` returns empty results despite a successful apply | The CLI is hitting real AWS instead of LocalStack | Add `--endpoint-url=http://aws:4566` or `http://localhost:4566` to the CLI call |
| `terraform apply` errors with `dial tcp: lookup aws on ...: no such host` | Terraform is being run outside the container network where `aws` resolves to LocalStack | Run from the lab's provided terminal, not a host shell |
| `Error: creating EC2 Key Pair: InvalidKeyPair.Duplicate` | A previous apply left the key pair in LocalStack | `terraform destroy` then re-apply, or `terraform import aws_key_pair.datacenter_kp datacenter-kp` |
| Apply succeeds but `.pem` file has wrong permissions | `file_permission` missing or formatted incorrectly | Set `file_permission = "0400"` exactly (string, four digits with leading zero) |
| AWS records `KeyType: ed25519` instead of `rsa` (on real AWS, not LocalStack) | Wrong algorithm in `tls_private_key` | Set `algorithm = "RSA"` (uppercase) |
| The grader fails despite everything looking correct | The grader checks for the file at `/home/bob/datacenter-kp.pem` at validation time; do not run `terraform destroy` before clicking validate | Re-run `terraform apply` and click validate before destroying |
