
# Fully Automated Infrastructure Setup with AWS CloudFormation

## Overview

This project demonstrates the complete provisioning of a **production-grade AWS environment** using only **Infrastructure as Code (IaC)** via **CloudFormation**. It includes automated deployment of network infrastructure (VPC), compute (EC2), database (RDS), and storage (S3), organized in **modular nested stacks**.

All resources are defined in reusable, parameterized templates for easy environment replication. Templates are version-controlled using Git and can be triggered in CI/CD workflows.

---

##  Project Goals

- Automate AWS infrastructure setup using only CloudFormation.
- Deploy reusable, modular templates with parameter overrides.
- Ensure environment consistency across dev, staging, and prod.
- Avoid manual configuration using declarative IaC principles.
- Enable CI/CD compatibility via version control and automation.

---

## Architecture 

- **GitHub**: Source code repository for CloudFormation templates.
- **AWS CodeBuild**: Package child templates to S3, Interpret cloudformation scripts, and programatically deploys updats to AWS infastructure stacks
- **Amazon S3**: An S3 bucket to store Child templates.
- **AWS CodePipeline**: Automates the entire CI/CD process.
- **AWS CloudFormation**: Provisions all resources in a repeatable and scalable manner.

---

## File Structure

```
├── master-stack.yaml               # Orchestrates nested stacks
├── templates/
│   ├── vpc.yaml                    # VPC infrastructure
│   ├── ec2.yaml                    # EC2 provisioning
│   ├── rds.yaml                    # RDS database
│   └── s3.yaml                     # S3 bucket
├── parameters/
│   ├── dev-params.json
│   └── prod-params.json
├── README.md
```


```
                     ┌──────────────┐
                     │ Master Stack │
                     └──────┬───────┘
                            │
 ┌────────────────────────────────────────────────┐
 │                 │               │              │
 ▼                 ▼               ▼              ▼
VPC Stack     EC2 Stack         S3 Stack       RDS Stack
   │             │                 │              │
   ▼             ▼                 ▼              ▼
  VPC       Security Group      S3 Bucket      DB Instance
 Subnets    EC2 Instance                       DB Subnets
 Routing                                       Secrets Manager
 IGW
```

---


## Infrastructure Components

### VPC Stack

- Custom VPC with public/private subnets
- Internet Gateway 
- Route Tables and associations

### EC2 Stack

- Amazon Linux EC2 instance in private subnet
- Security group with SSH/HTTP rules

### RDS Stack

- MySQL/PostgreSQL DB in private subnets
- Multi-AZ option enabled
- Parameter group, subnet group
- Secrets Manager integration 

### S3 Stack

- Versioned S3 bucket

---

## CI/CD Flow for Deployment

GitHub → CodePipeline → CodeBuild → CloudFormation Deployment (via master-stack.yaml)

- **Source**: GitHub push triggers CodePipeline
- **Build**:  Child templates are stored to s3, CloudFormation stacks are created/updated to reflect changes

---

## IAM & Security Considerations

- IAM roles are scoped using **least-privilege** principle.
- `CAPABILITY_NAMED_IAM` is used to create custom roles.
- EC2 and RDS are provisioned in **private subnets** with strict security group rules.
- Secrets such as DB credentials can be externalized using **AWS Secrets Manager**.

---

## Best Practices Followed

-  **Reusable Templates** – Modular and environment-agnostic
-  **Safe Deployments** – Used Change Sets for testing updates
-  **Code-Managed Infra** – Infrastructure stored in Git for full version history
-  **Automation-Ready** – Compatible with CI/CD pipelines (CodePipeline, GitHub Actions)

---


