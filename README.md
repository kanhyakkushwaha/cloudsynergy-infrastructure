# ☁️ CloudSynergy Core Infrastructure

> Production-ready AWS CloudFormation stack provisioning a secure, segmented VPC with an EC2 web instance — pipeline-safe and deployment-ready out of the box.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Parameters](#parameters)
- [Resources Provisioned](#resources-provisioned)
- [Deployment](#deployment)
- [Outputs](#outputs)
- [Security Considerations](#security-considerations)
- [Cost Estimate](#cost-estimate)
- [Cleanup](#cleanup)

---

## Overview

This CloudFormation template (`cloudsynergy-core.yaml`) provisions the **CloudSynergy Core Infrastructure** — a foundational AWS environment designed for web workloads. It creates an isolated VPC with clearly separated public and private subnets, internet connectivity via an Internet Gateway, and a hardened EC2 instance running Ubuntu 22.04.

The stack is:
- **Pipeline-safe** — parameterized and idempotent; safe to redeploy via CI/CD
- **Production-ready** — follows AWS Well-Architected Framework networking principles
- **Zero-SSH** — no SSH ingress exposed; access EC2 via AWS Systems Manager Session Manager

---

## Architecture

```
┌────────────────────────────────────────────────── ──┐
│                   CloudSynergy VPC                  │
│                   (10.0.0.0/16)                     │
│                                                     │
│  ┌──────────────────────┐  ┌──────────────────────┐ │
│  │   Public Subnet      │  │   Private Subnet     │ │
│  │   10.0.1.0/24  (AZ0) │  │   10.0.2.0/24  (AZ1) │ │
│  │                      │  │                      │ │
│  │  ┌────────────────┐  │  │   (Reserved for      │ │
│  │  │  EC2 Instance  │  │  │    future use)       │ │
│  │  │  Ubuntu 22.04  │  │  │                      │ │
│  │  │  Port 80 open  │  │  │                      │ │
│  │  └────────┬───────┘  │  │                      │ │
│  └───────────┼──────────┘  └──────────────────────┘ │
│              │                                      │
│  ┌───────────▼──────────┐                           │
│  │   Public Route Table │                           │
│  │   0.0.0.0/0 → IGW    │                           │
│  └───────────┬──────────┘                           │
└──────────────┼──────────────────────────────────────┘
               │
     ┌─────────▼─────────┐
     │  Internet Gateway │
     └─────────┬─────────┘
               │
           🌐 Internet
```

---

## Prerequisites

Before deploying, ensure you have:

- **AWS CLI** v2+ installed and configured (`aws configure`)
- **Permissions** to create: VPC, EC2, IAM-passable resources, CloudFormation stacks
- **AWS Region** selected that supports `t3.micro` (all standard regions do)
- An **S3 bucket** (optional) if uploading the template for large stack deployments

---

## Parameters

| Parameter      | Type   | Default    | Allowed Values          | Description              |
|----------------|--------|------------|-------------------------|--------------------------|
| `InstanceType` | String | `t3.micro` | `t2.micro`, `t3.micro`  | EC2 instance type to use |

---

## Resources Provisioned

| Logical ID                          | AWS Resource Type                      | Description                                  |
|-------------------------------------|----------------------------------------|----------------------------------------------|
| `CloudSynergyVPC`                   | `AWS::EC2::VPC`                        | VPC with DNS support enabled (10.0.0.0/16)   |
| `CloudSynergyIGW`                   | `AWS::EC2::InternetGateway`            | Internet Gateway for public internet access  |
| `AttachIGW`                         | `AWS::EC2::VPCGatewayAttachment`       | Attaches the IGW to the VPC                  |
| `PublicSubnet`                      | `AWS::EC2::Subnet`                     | Public subnet in AZ[0] (10.0.1.0/24)         |
| `PrivateSubnet`                     | `AWS::EC2::Subnet`                     | Private subnet in AZ[1] (10.0.2.0/24)        |
| `PublicRouteTable`                  | `AWS::EC2::RouteTable`                 | Route table for the public subnet            |
| `PublicRoute`                       | `AWS::EC2::Route`                      | Default route (0.0.0.0/0) via IGW            |
| `PublicSubnetRouteTableAssociation` | `AWS::EC2::SubnetRouteTableAssociation`| Associates the public subnet to its RT       |
| `WebSecurityGroup`                  | `AWS::EC2::SecurityGroup`              | Allows inbound HTTP (port 80) only           |
| `CloudSynergyEC2`                   | `AWS::EC2::Instance`                   | Ubuntu 22.04 EC2 in the public subnet        |

---

## Deployment

### Option 1 — AWS CLI

```bash
aws cloudformation deploy \
  --template-file cloudsynergy-core.yaml \
  --stack-name cloudsynergy-core \
  --parameter-overrides InstanceType=t3.micro \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

### Option 2 — AWS Console

1. Navigate to **CloudFormation → Stacks → Create Stack**
2. Choose **Upload a template file** and select `cloudsynergy-core.yaml`
3. Set the stack name to `cloudsynergy-core`
4. Select your preferred `InstanceType`
5. Review and click **Create Stack**

### Option 3 — CI/CD Pipeline (GitHub Actions example)

```yaml
- name: Deploy CloudSynergy Stack
  run: |
    aws cloudformation deploy \
      --template-file cloudsynergy-core.yaml \
      --stack-name cloudsynergy-core \
      --parameter-overrides InstanceType=t3.micro \
      --region ${{ secrets.AWS_REGION }} \
      --no-fail-on-empty-changeset
```

---

## Outputs

After a successful deployment, the following values are exported:

| Output Key    | Description                          |
|---------------|--------------------------------------|
| `VPCId`       | The ID of the provisioned VPC        |
| `EC2PublicIP` | The public IP address of the EC2 instance |

### Retrieve outputs via CLI

```bash
aws cloudformation describe-stacks \
  --stack-name cloudsynergy-core \
  --query "Stacks[0].Outputs" \
  --output table
```

---

## Security Considerations

| Area              | Status  | Notes                                                                 |
|-------------------|---------|-----------------------------------------------------------------------|
| SSH Access        | ✅ Disabled | No port 22 in the Security Group — use SSM Session Manager instead |
| HTTP Access       | ⚠️ Open | Port 80 is open to `0.0.0.0/0` — restrict to known CIDRs in production |
| HTTPS             | ❌ Not configured | Add an ACM cert + ALB or NGINX termination for TLS in production |
| AMI Source        | ✅ Dynamic | AMI resolved via SSM Parameter Store — always uses latest Ubuntu 22.04 |
| Private Subnet    | ℹ️ Reserved | No NAT Gateway provisioned; add one before deploying private workloads |

> **Recommendation**: Before going to production, restrict `CidrIp` on port 80 to a load balancer CIDR or CloudFront prefix list, and add an HTTPS listener.

---

## Cost Estimate

> Estimates based on `us-east-1` on-demand pricing. Actual costs may vary by region.

| Resource           | Approx. Monthly Cost   |
|--------------------|------------------------|
| EC2 `t3.micro`     | ~$7.59 / month         |
| VPC / Subnets      | Free                   |
| Internet Gateway   | Free (data transfer billed separately) |
| **Total (idle)**   | **~$7.59 / month**     |

---

## Cleanup

To avoid ongoing charges, delete the stack when no longer needed:

```bash
aws cloudformation delete-stack \
  --stack-name cloudsynergy-core \
  --region us-east-1
```

Verify deletion:

```bash
aws cloudformation describe-stacks \
  --stack-name cloudsynergy-core
```

---

## License

This project is proprietary to **CloudSynergy**. All rights reserved.
