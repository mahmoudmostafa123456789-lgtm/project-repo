# Handover checklist

- [ ] Confirm AWS account and target region with recipient.
- [ ] Replace the remote-state bucket placeholder in both backend.hcl files.
- [ ] Bootstrap the S3 state bucket and DynamoDB lock table.
- [ ] Run `terraform fmt -recursive` and `terraform validate` in bootstrap and each environment.
- [ ] Review `terraform plan` for dev; deploy only after approval.
- [ ] Confirm HTTP page loads and SSM Session Manager works.
- [ ] Review costs and destroy test resources after demonstration.
- [ ] Transfer repository ownership and document any account-specific values.

## Important limitation
This package is source code and documentation only. It has not been applied to an AWS account, and no live infrastructure, credentials, Git remote, or walkthrough recording is included.
