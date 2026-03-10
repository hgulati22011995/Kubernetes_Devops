
---

# AWS VPC Peering (Cross-Region) – Terraform Demo

## Overview

This repository demonstrates **cross-region AWS VPC peering** using **Terraform**.
It provisions **two isolated VPCs in different AWS regions**, establishes **VPC peering between them**, configures **routing and security**, and deploys **EC2 instances** to validate **private network connectivity** across regions.

This is **not a toy setup**. It mirrors how real-world teams validate inter-VPC communication without using public internet paths.

---

## What This Project Proves

If you claim AWS networking experience, you should understand and be able to explain:

* How **VPC peering works across regions**
* Why **route tables** matter more than the peering connection itself
* How **security groups + CIDR scoping** control east-west traffic
* Why peering **does not support transitive routing**
* How to **validate private connectivity** (not just “Terraform apply succeeded”)

This repo forces you to confront all of that.

---

## Architecture Summary

```
┌────────────────────────┐         VPC Peering        ┌────────────────────────┐
│  Primary VPC           │ <───────────────────────> │  Secondary VPC         │
│  Region: us-east-1     │                           │  Region: us-west-2     │
│  CIDR: 10.0.0.0/16     │                           │  CIDR: 10.1.0.0/16     │
│                        │                           │                        │
│  EC2 Instance          │                           │  EC2 Instance          │
│  Private IP            │                           │  Private IP            │
└────────────────────────┘                           └────────────────────────┘
```

**Traffic Flow**

* EC2 (Primary) → Route Table → VPC Peering → Route Table → EC2 (Secondary)
* No NAT
* No Internet traversal
* Pure private routing

---

## Resources Created

### Networking

* 2 × VPCs (cross-region)
* 2 × Public subnets
* 2 × Internet Gateways
* 2 × Route tables
* 1 × VPC peering connection (requester + accepter)
* Explicit peering routes in **both** VPCs

### Compute

* 2 × EC2 instances (Ubuntu)
* Apache installed via `user_data`
* Instances expose private IP for verification

### Security

* Security groups allowing:

  * SSH (for demo access)
  * ICMP (ping) between VPC CIDRs
  * TCP traffic between VPC CIDRs

---

## Repository Structure

```
.
├── main.tf            # Core infrastructure (VPCs, peering, EC2)
├── provider.tf        # Multi-region provider configuration
├── local.tf           # User-data scripts for EC2
├── outputs.tf         # Verification outputs
├── terraform.tfvars   # Environment-specific values
└── variables.tf       # Input variables
```

---

## Prerequisites (Read This Carefully)

If you skip any of these, the deployment will fail—and that’s on you.

* Terraform **>= 1.6.0**
* AWS CLI configured with valid credentials
* IAM permissions to create:

  * VPCs, Subnets, Routes
  * EC2 instances
  * VPC Peering Connections
* **EC2 key pairs created in both regions**

  * Key names must match `terraform.tfvars`

---

## Configuration

Update `terraform.tfvars`:

```hcl
primary_region   = "us-east-1"
secondary_region = "us-west-2"

primary_vpc_cidr   = "10.0.0.0/16"
secondary_vpc_cidr = "10.1.0.0/16"

primary_key_name   = "vpc-peering-demo"
secondary_key_name = "vpc-peering-demo"
```

**Do not reuse CIDR blocks.**
Overlapping CIDRs = broken networking. Period.

---

## Deployment Steps

```bash
terraform init
terraform validate
terraform plan
terraform apply
```

Read the plan.
If you blindly apply infrastructure, you’re not a DevOps engineer—you’re a risk.

---

## Verification (This Is the Whole Point)

Terraform succeeding means **nothing** unless traffic flows.

### Option 1: Primary → Secondary

```bash
ssh -i your-key.pem ubuntu@<PRIMARY_PUBLIC_IP>
ping <SECONDARY_PRIVATE_IP>
```

### Option 2: Secondary → Primary

```bash
ssh -i your-key.pem ubuntu@<SECONDARY_PUBLIC_IP>
ping <PRIMARY_PRIVATE_IP>
```

**Expected Result:**

* ICMP succeeds
* Traffic stays on private IPs

If ping fails:

* Check route tables
* Check security group CIDR rules
* Check peering status

---

## Outputs Provided

Terraform exposes:

* VPC IDs
* CIDR blocks
* Peering connection ID & status
* EC2 public and private IPs
* Ready-to-use connectivity test commands

These outputs exist so you **don’t guess**—you verify.

---

## Key Limitations (Don’t Ignore These)

* ❌ No transitive routing
* ❌ No overlapping CIDRs
* ❌ No security group referencing across regions
* ❌ Peering does NOT scale like Transit Gateway

If you try to build hub-and-spoke with peering, you’re designing it wrong.

---

## When to Use VPC Peering (And When Not To)

### Use It When:

* Low-latency, private communication
* Simple, point-to-point VPC connectivity
* Minimal network sprawl

### Don’t Use It When:

* You need transitive routing
* You have more than a handful of VPCs
* You want centralized inspection or firewalling

That’s when **Transit Gateway** exists.

---

## Cleanup (Don’t Leave Bills Running)

```bash
terraform destroy
```

Always destroy demo infrastructure.
Leaving it running is how cloud bills destroy careers.

---

## Final Reality Check

This project is **resume-worthy** *only if* you can explain:

* Why each route exists
* Why each CIDR rule exists
* What breaks if one component is removed

If you can’t explain it, you don’t understand it—no matter how clean the code looks.

---


