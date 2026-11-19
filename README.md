# Small AWS Infrastructure with Terraform

A reusable AWS foundation with a VPC, public/private subnets in two Availability Zones, an optional NAT Gateway, a small Amazon Linux EC2 web server, and a private S3 bucket. The EC2 instance is managed through AWS Systems Manager (SSM); **SSH ingress is not opened**.

## Architecture

- One VPC with DNS support.
- Two public and two private subnets across two AZs.
- Internet Gateway and public route tables.
- Optional single NAT Gateway for private-subnet outbound internet access (`enable_nat_gateway`, off by default because NAT has an hourly/data charge).
- EC2 web instance in a public subnet, with a public IPv4 address, HTTP ingress only, and SSM-managed access (no SSH).
- Encrypted, versioned S3 bucket with public access blocked and TLS-only bucket policy.
- IAM instance role with SSM managed-instance permissions.
- Terraform modules, separate `dev` and `prod` variable files, and remote-state bootstrap.

> Cost note: AWS resources can incur charges. NAT Gateway, EC2, EBS, S3, and public IPv4 may be billable. Review the AWS Pricing Calculator and destroy resources when finished. This is a starter deployment, not a compliance certification.

## Repository layout

```text
.
├── bootstrap/              # Creates S3 + DynamoDB remote-state resources
├── modules/
│   ├── network/
│   ├── security/
│   ├── compute/
│   └── storage/
├── environments/
│   ├── dev/
│   └── prod/
└── README.md
```

## Prerequisites

- Terraform >= 1.6
- AWS CLI v2 configured with credentials for the target account
- IAM permissions to create VPC, EC2, IAM, S3, and (when enabled) NAT resources
- An AWS region supported by the selected instance type

Check credentials:

```bash
aws sts get-caller-identity
terraform version
```

## 1. Bootstrap remote state (one-time per AWS account/region)

The backend bucket must exist before Terraform can initialize the main stack. The bootstrap stack uses local state initially.

```bash
cd bootstrap
terraform init
terraform apply \
  -var="state_bucket_name=YOUR-GLOBALLY-UNIQUE-STATE-BUCKET" \
  -var="lock_table_name=small-aws-terraform-locks"
```

Record the bucket name. Do not delete this bucket while any environment uses it.

## 2. Configure an environment

For dev:

```bash
cd ../environments/dev
```

For prod:

```bash
cd ../prod
```

Edit `backend.hcl` and replace `REPLACE_WITH_YOUR_STATE_BUCKET`. The backend key is already environment-specific. Then initialize:

```bash
terraform init -backend-config=backend.hcl
terraform plan -var-file=terraform.tfvars
terraform apply -var-file=terraform.tfvars
```

Use a unique `project_name` and review the plan before applying. The provided dev/prod files use distinct CIDRs and names. `prod` defaults to two EC2 instances for basic redundancy; adjust instance count and sizing to your needs.

## 3. Connect to the instance

Get the instance IDs:

```bash
terraform output instance_ids
```

Open **AWS Console → Systems Manager → Session Manager**, select an instance, and start a session. Or use:

```bash
aws ssm start-session --target INSTANCE_ID
```

The instance profile includes `AmazonSSMManagedInstanceCore`. The instance is in a public subnet so it can reach SSM public endpoints without a NAT Gateway. Its security group does not allow SSH.

## 4. Destroy resources

Destroy an environment first:

```bash
terraform destroy -var-file=terraform.tfvars
```

Repeat for the other environment if deployed. Keep the backend resources until all state is no longer needed. To remove the bootstrap resources last:

```bash
cd ../../bootstrap
terraform destroy \
  -var="state_bucket_name=YOUR-GLOBALLY-UNIQUE-STATE-BUCKET" \
  -var="lock_table_name=small-aws-terraform-locks"
```

The S3 backend bucket has versioning enabled; Terraform state versions may remain billable. Confirm you have exported/retained any state you need before removing it.

## Configuration notes

- `enable_nat_gateway = false` by default to avoid NAT charges. If private workloads need outbound internet, set it to `true`; the module creates one NAT Gateway in the first public subnet and routes both private subnets through it (single-AZ NAT is cheaper but not highly available).
- `instance_count` controls EC2 count.
- `allowed_http_cidrs` defaults to `0.0.0.0/0` for a public demo page. Restrict this to trusted CIDRs for non-public workloads.
- The S3 bucket name is generated from the project name and AWS account ID unless overridden.
- RDS is intentionally not provisioned because it was optional in the request and introduces additional cost. Add it as a separate module when database engine, sizing, backup, and network requirements are known.

## Security practices included

- No inbound SSH.
- EC2 administration through SSM and an instance role.
- IMDSv2 required.
- Encrypted EBS and S3.
- S3 public access block, versioning, TLS-only policy.
- Least-privilege custom S3 policy is not attached to EC2 because the sample web server does not need bucket access.
- Environment-specific state keys and CIDR ranges.

## Handover walkthrough (suggested 10 minutes)

1. Explain the VPC, subnet and route-table layout.
2. Review security groups and the no-SSH / SSM access pattern.
3. Show the S3 bucket controls.
4. Run `terraform plan` for dev and explain the environment variables.
5. Show outputs and an SSM Session Manager connection.
6. Demonstrate `terraform destroy` and explain remote-state retention.

## Before production use

Add a formal review of IAM permissions, network exposure, tagging, budgets/alerts, CloudTrail organization-level coverage, backup/restore tests, monitoring/alerting, and multi-AZ NAT design. Pin provider/module versions and review upgrades. Do not commit credentials, `.tfstate`, `.tfvars` containing secrets, or local Terraform directories.
