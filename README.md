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
├── master-stack.yaml           # Orchestrates nested stacks
├── vpc.yaml                    # VPC infrastructure
├── ec2.yaml                    # EC2 provisioning
├── rds.yaml                    # RDS database
├── s3.yaml                     # S3 bucket
├── prod-params.json            # JSON file with parameters
├── buildspec.yml               # Codebuild file for deployments
```


![GitHub](/iac/Screenshot4.png)

**CODE**

```
AWSTemplateFormatVersion: '2010-09-09'

Parameters:
  Environment:
    Type: String

  VpcCIDR:
    Type: String

  PublicSubnet1CIDR:
    Type: String

  PublicSubnet2CIDR:
    Type: String

  PrivateSubnet1CIDR:
    Type: String

  PrivateSubnet2CIDR:
    Type: String

  AMI:
    Type: String

  DBUser:
    Type: String

  DBPassword:
    Type: String

Resources:
  VPCStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/iac-bucket-azamcloud/vpc.yaml
      Parameters:
        Environment: !Ref Environment
        VpcCIDR: !Ref VpcCIDR
        PublicSubnet1CIDR: !Ref PublicSubnet1CIDR
        PublicSubnet2CIDR: !Ref PublicSubnet2CIDR
        PrivateSubnet1CIDR: !Ref PrivateSubnet1CIDR
        PrivateSubnet2CIDR: !Ref PrivateSubnet2CIDR



  EC2Stack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/iac-bucket-azamcloud/ec2.yaml
      Parameters:
        VPCOut: !GetAtt VPCStack.Outputs.VPCOut
        PublicSubnet1Out: !GetAtt VPCStack.Outputs.PublicSubnet1Out
        AMI: !Ref AMI

  RDSStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/iac-bucket-azamcloud/rds.yaml
      Parameters:
        SecurityGroupOut: !GetAtt EC2Stack.Outputs.SecurityGroupOut
        PrivateSubnet1Out: !GetAtt VPCStack.Outputs.PrivateSubnet1Out
        PrivateSubnet2Out: !GetAtt VPCStack.Outputs.PrivateSubnet2Out
        DBUser: !Ref DBUser
        DBPassword: !Ref DBPassword

        
  S3Stack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/iac-bucket-azamcloud/s3.yaml
      Parameters:
        Environment: !Ref Environment
```

![GitHub](/iac/Screenshot8.png)

**CODE**

```
AWSTemplateFormatVersion: '2010-09-09'

Parameters:
  Environment:
    Type: String

  VpcCIDR:
    Type: String

  PublicSubnet1CIDR:
    Type: String

  PublicSubnet2CIDR:
    Type: String

  PrivateSubnet1CIDR:
    Type: String

  PrivateSubnet2CIDR:
    Type: String

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Ref Environment

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Ref Environment

  InternetGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref VPC

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [ 0, !GetAZs '' ]
      CidrBlock: !Ref PublicSubnet1CIDR
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub ${Environment} Public Subnet (AZ1)

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [ 1, !GetAZs  '' ]
      CidrBlock: !Ref PublicSubnet2CIDR
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub ${Environment} Public Subnet (AZ2)

  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [ 0, !GetAZs  '' ]
      CidrBlock: !Ref PrivateSubnet1CIDR
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: !Sub ${Environment} Private Subnet (AZ1)

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [ 1, !GetAZs  '' ]
      CidrBlock: !Ref PrivateSubnet2CIDR
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: !Sub ${Environment} Private Subnet (AZ2)

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub ${Environment} Public Routes

  DefaultPublicRoute:
    Type: AWS::EC2::Route
    DependsOn: InternetGatewayAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet1

  PublicSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet2


  PrivateRouteTable1:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub ${Environment} Private Routes

  PrivateSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable1
      SubnetId: !Ref PrivateSubnet1

  PrivateSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable1
      SubnetId: !Ref PrivateSubnet2

  NoIngressSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: "no-ingress-sg"
      GroupDescription: "Security group with no ingress rule"
      VpcId: !Ref VPC

Outputs:
  VPCOut:
    Description: A reference to the created VPC
    Value: !Ref VPC

  PublicSubnetsOut:
    Description: A list of the public subnets
    Value: !Join [ ",", [ !Ref PublicSubnet1, !Ref PublicSubnet2 ]]

  PrivateSubnetsOut:
    Description: A list of the private subnets
    Value: !Join [ ",", [ !Ref PrivateSubnet1, !Ref PrivateSubnet2 ]]

  PublicSubnet1Out:
    Description: A reference to the public subnet in the 1st Availability Zone
    Value: !Ref PublicSubnet1

  PublicSubnet2Out:
    Description: A reference to the public subnet in the 2nd Availability Zone
    Value: !Ref PublicSubnet2

  PrivateSubnet1Out:
    Description: A reference to the private subnet in the 1st Availability Zone
    Value: !Ref PrivateSubnet1

  PrivateSubnet2Out:
    Description: A reference to the private subnet in the 2nd Availability Zone
    Value: !Ref PrivateSubnet2

  NoIngressSecurityGroupOut:
    Description: Security group with no ingress rule
    Value: !Ref NoIngressSecurityGroup
```

![GitHub](/iac/Screenshot3.png)

**CODE**

```
AWSTemplateFormatVersion: '2010-09-09'

Parameters:
  VPCOut:
    Type: String
  PublicSubnet1Out:
    Type: String
  AMI:
    Type: String

Resources:
  SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow SSH and HTTP
      VpcId: !Ref VPCOut
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          CidrIp: 0.0.0.0/0 

  EC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref AMI
      VpcId: !Ref VPCOut
      InstanceType: t3.micro
      SubnetId: !Ref PublicSubnet1Out
      SecurityGroupIds:
        - !Ref SecurityGroup

Outputs:
  SecurityGroupOut:
    Description: A reference to the created VPC
    Value: !Ref SecurityGroup
```

![GitHub](/iac/Screenshot6.png)

**CODE**

```
AWSTemplateFormatVersion: '2010-09-09'

Parameters:
  DBUser:
    Type: String
  DBPassword:
    Type: String
  SecurityGroupOut:
    Type: String
  PrivateSubnet1Out:
    Type: String
  PrivateSubnet2Out: 
    Type: String

Resources:

  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: "Subnet group for RDS"
      SubnetIds:
        - !Ref PrivateSubnet1Out
        - !Ref PrivateSubnet2Out
      Tags:
        - Key: Name
          Value: "RDS-Subnet-Group"

  RDSInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceClass: db.t3.micro
      Engine: mysql
      AllocatedStorage: 20
      DBSubnetGroupName: !Ref DBSubnetGroup
      VPCSecurityGroups:
        - !Ref SecurityGroupOut
      MasterUsername: !Ref DBUser
      MasterUserPassword: !Ref DBPassword
      PubliclyAccessible: false
      Tags:
        - Key: Name
          Value: "My-RDS-Instance"
```

![GitHub](/iac/Screenshot7.png)

**CODE**

```
AWSTemplateFormatVersion: '2010-09-09'

Parameters:
  Environment:
    Type: String

Resources:
  AppDataBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "${Environment}-app-data-${AWS::AccountId}"
      VersioningConfiguration:
        Status: Enabled
```

![GitHub](/iac/Screenshot5.png)

**CODE**

```
[
  {
    "ParameterKey": "Environment",
    "ParameterValue": "prod"
  },
  {
    "ParameterKey": "VpcCIDR",
    "ParameterValue": "10.0.0.0/16"
  },
  {
    "ParameterKey": "PublicSubnet1CIDR",
    "ParameterValue": "10.0.1.0/24"
  },
  {
    "ParameterKey": "PublicSubnet2CIDR",
    "ParameterValue": "10.0.2.0/24"
  },
  {
    "ParameterKey": "PrivateSubnet1CIDR",
    "ParameterValue": "10.0.3.0/24"
  },
  {
    "ParameterKey": "PrivateSubnet2CIDR",
    "ParameterValue": "10.0.4.0/24"
  },
  {
    "ParameterKey": "AMI",
    "ParameterValue": "ami-0150ccaf51ab55a51"
  },
  {
    "ParameterKey": "DBUser",
    "ParameterValue": "admin"
  },
  {
    "ParameterKey": "DBPassword",
    "ParameterValue": "cnhnhbejhfbe857748"
  }
]
```

![GitHub](/iac/Screenshot2.png)

**CODE**

```
version: 0.2

phases:
  install:
    runtime-versions:
      python: 3.11

  build:
    commands:
      - echo Packaging templates...
      - aws s3 cp vpc.yaml s3://iac-bucket-azamcloud/
      - aws s3 cp ec2.yaml s3://iac-bucket-azamcloud/
      - aws s3 cp rds.yaml s3://iac-bucket-azamcloud/
      - aws s3 cp s3.yaml s3://iac-bucket-azamcloud/
      - aws cloudformation deploy --template-file master-stack.yaml --stack-name iac-prod --capabilities CAPABILITY_NAMED_IAM --parameter-overrides file://prod-params.json

artifacts:
  files:
    - '**/*'
```


```
                     ┌────────-─────┐
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

## CI/CD Flow for Deployment

GitHub → CodePipeline → CodeBuild → CloudFormation Deployment (via master-stack.yaml)

- **Source**: GitHub push triggers CodePipeline
- **Build**:  Child templates are stored to s3, CloudFormation stacks are created/updated to reflect changes

![CI/CD](/iac/Screenshot1.png)
![CI/CD](/iac/Screenshot18.png)
![CI/CD](/iac/Screenshot19.png)
![CI/CD](/iac/Screenshot9.png)
![CI/CD](/iac/Screenshot10.png)
![CI/CD](/iac/Screenshot12.png)
![CI/CD](/iac/Screenshot13.png)
![CI/CD](/iac/Screenshot11.png)

---

## Infrastructure Components

### VPC Stack

- Custom VPC with public/private subnets
- Internet Gateway 
- Route Tables and associations

![Infra](/iac/Screenshot14.png)

### EC2 Stack

- Amazon Linux EC2 instance in private subnet
- Security group with SSH/HTTP rules

![Infra](/iac/Screenshot16.png)

### RDS Stack

- MySQL/PostgreSQL DB in private subnets
- Multi-AZ option enabled
- Parameter group, subnet group
- Secrets Manager integration 

![Infra](/iac/Screenshot17.png)

### S3 Stack

- Versioned S3 bucket

![Infra](/iac/Screenshot15.png)

---

## Best Practices Followed

-  **Reusable Templates** – Modular and environment-agnostic
-  **Safe Deployments** – Used Change Sets for testing updates
-  **Code-Managed Infra** – Infrastructure stored in Git for full version history
-  **Automation-Ready** – Compatible with CI/CD pipelines (CodePipeline, GitHub Actions)

---


