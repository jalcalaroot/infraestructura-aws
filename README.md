# 🚀 AWS Enterprise Infrastructure Automator

[![CloudFormation](https://img.shields.io/badge/IaC-CloudFormation-orange?logo=amazon-aws)](https://aws.amazon.com/cloudformation/)
[![Security](https://img.shields.io/badge/Security-Checkov-brightgreen?logo=bridgecrew)](https://www.checkov.io/)
[![CI/CD](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-blue?logo=github-actions)](https://github.com/features/actions)

Modular CloudFormation infrastructure-as-code, deployed via a single
reusable GitHub Actions pipeline (OIDC auth, cfn-lint, Checkov, drift
detection, change sets). Four independent stacks build up a small
Active Directory + WorkSpaces environment on top of a hardened VPC.

## Architecture

```
vpc/          -> VPC, 3 AZs x 4 subnet tiers, NAT GW, Flow Logs, 11 VPC endpoints
   |
   +--> bastionhost/  -> EC2 bastion (SSM Session Manager access)
   |
   +--> directory/    -> Simple AD + Secrets Manager secret
              |
              +--> workspaces/  -> WorkSpaces desktop, joined to the directory
```

### `vpc/` - VPC-Enterprise-3AZ-Hardened

- 1 VPC (`/16`, size via `ClassA`/`ClassB`) across 3 AZs, 4 subnet tiers each
  (Public, Private, Intra, TGW-reserved) = 12 subnets.
- 1 Internet Gateway, 1 **regional** NAT Gateway (not per-AZ - a
  cost/HA tradeoff, see Security notes).
- VPC Flow Logs (all traffic) to a KMS-encrypted CloudWatch Log Group
  (customer-managed key, rotation enabled).
- 2 Gateway endpoints (S3, DynamoDB) + 9 Interface endpoints (EC2, SSM,
  SSMMessages, EC2Messages, KMS, Logs, Secrets Manager, ECR x2) - lets
  private-subnet resources reach AWS APIs without a NAT/IGW hop.
- Exports (`Fn::ImportValue`-able): VPC ID/CIDR, subnet ID lists per
  tier, endpoint security group ID, intra route table ID.

### `bastionhost/` - EC2 bastion

- EC2 instance (default `t3.micro`) in the first private subnet, IAM
  role with `AmazonSSMManagedInstanceCore` (Session Manager access, no
  key pair needed), encrypted gp3 root volume.
- Imports the VPC stack's exports - **must be deployed after `vpc/`**.

### `directory/` - Simple AD

- `AWS::DirectoryService::SimpleAD` ("Small"), in the first two private
  subnets.
- Admin password stored both as a `NoEcho` CloudFormation parameter and
  mirrored into a Secrets Manager secret (AWS-managed key) for later
  reference/rotation.
- Exports the Directory ID for the WorkSpaces stack.

### `workspaces/` - WorkSpaces desktop

- One `AWS::WorkSpaces::Workspace`, joined to the directory from the
  previous stack - **must be deployed after `directory/`**.
- Root + user volumes encrypted with the AWS-managed `alias/aws/workspaces` key.
- `RunningMode: AUTO_STOP` (60 min idle timeout) - the main cost control
  lever for this stack.

## Prerequisites

- An AWS account with an OIDC identity provider trusting GitHub Actions,
  and a role ARN with permissions to deploy these stacks
  (`CAPABILITY_NAMED_IAM`). Set it as the `AWS_ROLE_ARN` repository secret.
- No long-lived AWS access keys are used anywhere in this repo - auth is
  OIDC federation (`aws-actions/configure-aws-credentials`).

## Deploy order

**`vpc` → `directory` → {`bastionhost`, `workspaces`}**

Each stack has its own workflow (`deploy-vpc.yml`, `deploy-bastionhost.yml`,
`deploy-directory.yml`, `deploy-workspaces.yml`), triggered by pushes that
touch that stack's folder, or manually via `workflow_dispatch`. **Deploy
order is a convention, not enforced by CI** - the workflows don't chain or
wait on each other, so a first-time deploy of `directory/`, `bastionhost/`,
or `workspaces/` before `vpc/` will fail on missing `Fn::ImportValue`
exports.

## Teardown

Reverse order: `workspaces` → `bastionhost`/`directory` → `vpc` (later
stacks import the VPC/directory's exports, so they must go first).
Trigger each stack's workflow manually, or delete the CloudFormation
stacks directly with the AWS CLI/console.

## Parameters

Each stack folder has its own `parameters.json` with example values
(`ProjectName`, `Environment`, `Owner`, `CostCenter`, plus stack-specific
ones like `directory/`'s `ADAdminPassword` or `workspaces/`'s `BundleId`).
Edit these before deploying into your own account - in particular,
`workspaces/parameters.json`'s `BundleId` is a WorkSpaces bundle ID that's
specific to a region/account and must be verified for yours, and
`directory/parameters.json`'s `ADAdminPassword` placeholder must be
replaced with a real value (or wired to a dynamic Secrets Manager
reference - see Security notes).

## Pipeline (`main.yml`, called by all 4 `deploy-*.yml`)

1. **Gitleaks** secret scan (fails the pipeline on a hit).
2. **AWS auth** via OIDC (`AWS_ROLE_ARN`).
3. **cfn-lint** syntax validation.
4. **Checkov** security scan (soft-fail - reports Critical/High/Medium
   counts to the job summary but doesn't block the pipeline).
5. **Stack inspection**: handles `ROLLBACK_COMPLETE` recovery, detects
   create vs. update, runs `detect-stack-drift` on updates.
6. **Deploy via CloudFormation change sets** (`CAPABILITY_NAMED_IAM`).

## Security notes

- **Bastion security group allows SSH (22) from `0.0.0.0/0`.** SSM
  Session Manager is already wired up via the VPC's interface endpoints,
  making the open SSH port redundant attack surface - restrict this or
  drop the SSH rule entirely in favor of SSM-only access.
- **AD admin password is a `NoEcho` CloudFormation parameter**, not a
  Secrets Manager dynamic reference - the `directory/template.yaml`
  cfn-lint pragma that would normally flag this (W1011) is suppressed at
  the pipeline level. It's masked in the console/CLI but still passed as
  plaintext at deploy time via `parameters.json`.
- **Single (non-per-AZ) NAT Gateway** in the VPC stack - cost-optimized,
  but a single point of failure for private-subnet egress.
- Secrets Manager secret and WorkSpaces volumes use AWS-managed keys, not
  a customer-managed CMK.

## Files

| Folder | Contents |
|---|---|
| `vpc/` | `template.yaml`, `parameters.json` |
| `bastionhost/` | `template.yaml`, `parameters.json` |
| `directory/` | `template.yaml`, `parameters.json` |
| `workspaces/` | `template.yaml`, `parameters.json` |
| `.github/workflows/main.yml` | Reusable deploy engine (lint, security scan, drift, change set) |
| `.github/workflows/deploy-*.yml` | Per-stack triggers, calling `main.yml` |
