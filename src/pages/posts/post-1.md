---
layout: ../../layouts/PostLayout.astro
title: Building and Deploying Efficient Infrastructure for Auditrol with Pulumi 🚀
# pubDate: 2022-07-01
pubDate: April 03, 2025
description: "This is the first post of my new Astro blog."
author: Teebra Deka
image:
  url: "https://auditrol.com/static/images/webp/Auditrol_Pulumi.webp"
  alt: "The Astro logo on a dark background with a pink glow."
tags: ["astro", "blogging", "learning in public"]
---

Imagine spinning up your cloud infrastructure as easily as writing a few lines of code in your favorite programming language. Sounds pretty great, right? That’s exactly what Pulumi, a modern Infrastructure as Code (IaC) tool, brings to the table. Unlike traditional tools that rely on specialized languages or bulky templates, Pulumi lets you define your cloud resources using languages you already know such as Python, TypeScript, or Go. This means your infrastructure code becomes more readable and reusable across cloud infrastructure.

In this blog, we’ll walk you through a practical, hands-on example: provisioning an AWS setup that includes a **Virtual Private Cloud (VPC)**, a **Relational Database Service (RDS)** instance, and an **Elastic Kubernetes Service (EKS)** cluster. By the end, you’ll see how Pulumi simplifies cloud management and empowers you to build efficient, scalable infrastructure with ease.

Ready? Let’s get started!

## 🌟 Why Pulumi? Code Your Cloud, Your Way

Traditional IaC tools like Terraform or CloudFormation have limitations:

- **Static configuration files** that lack flexibility.
- **No loops, conditionals, or reusability** without complex workarounds.
- **Steep learning curves** for domain-specific languages (HCL/YAML).

Pulumi solves these issues by letting you:  
✅ **Use Python, JavaScript, Go, or C#** to define infrastructure.  
✅ **Reuse code** with functions, classes, and packages.  
✅ **Integrate with CI/CD pipelines** effortlessly.  
✅ **Manage state securely** via Pulumi’s backend or your own storage.

_Think of it as infrastructure automation, supercharged._

## 🛠️ Getting Started: Install Pulumi & Configure AWS

### Step 1: Install Pulumi

Run this one-liner in your terminal:

```bash
curl -fsSL https://get.pulumi.com | sh
```

_Already use Homebrew?_ `brew install pulumi` works too!

_This downloads and installs Pulumi. Once it’s done, you’re halfway there!_

### Step 2: Set Up AWS Credentials

Ensure the AWS CLI is installed and configured. If you haven’t installed the AWS CLI yet, do that first (check [AWS’s guide](https://aws.amazon.com/cli/) if needed). Then run:

```bash
aws configure
```

_Enter your AWS Access Key and Secret Key when prompted. That’s it—Pulumi will use these to manage your AWS resources._

### Step 3: Create a New Pulumi Project

Now, let’s kick off a new project. We’ll use Python for this example (but feel free to pick your preferred language). Run:

```bash
pulumi new aws-python
```

_This sets up a new directory with all the project files you need. It’ll also ask you to log in to Pulumi if you haven’t already—super easy!_

With the setup out of the way, let’s start building our AWS infrastructure.

## 🏗️ Building Your AWS Infrastructure

Let’s define three core components: a **VPC**, an **RDS** database, and an **EKS Cluster**. I’ll break it down step-by-step with full code examples and explanations so you can follow along.

#### **1. Create a VPC**

First up, we’ll build a Virtual Private Cloud—a private network space for all our resources. Here’s how to do it:

```python
import pulumi
import pulumi_aws as aws

# Create a new VPC
vpc = aws.ec2.Vpc("custom-vpc",
    cidr_block="10.0.0.0/16",
    enable_dns_hostnames=True,
    tags={"Name": "custom-vpc"})

# Add two subnets for high availability
subnet1 = aws.ec2.Subnet("subnet1",
    vpc_id=vpc.id,
    cidr_block="10.0.1.0/24",
    availability_zone="us-west-2a")

subnet2 = aws.ec2.Subnet("subnet2",
    vpc_id=vpc.id,
    cidr_block="10.0.2.0/24",
    availability_zone="us-west-2b")
```

**What’s happening here?**

- The VPC gets a `10.0.0.0/16` CIDR block, giving us plenty of IP addresses to work with.
- We enable DNS hostnames so our resources can have friendly names.
- Two subnets in different availability zones (`us-west-2a` and `us-west-2b`) ensure our setup is resilient.

#### **2. Deploy an RDS Database**

Next, let’s set up a PostgreSQL database using AWS RDS. We’ll tie it to our VPC and add some security:

```python
# Create a subnet group for the RDS instance
db_subnet_group = aws.rds.SubnetGroup("db-subnet-group",
    subnet_ids=[subnet1.id, subnet2.id])

# Set up a security group to control access
rds_sg = aws.ec2.SecurityGroup("rds-sg",
    vpc_id=vpc.id,
    description="Allow PostgreSQL traffic",
    ingress=[{
        "protocol": "tcp",
        "from_port": 5432,
        "to_port": 5432,
        "cidr_blocks": ["0.0.0.0/0"]  # Open for demo; restrict in production!
    }])

# Deploy the RDS instance
rds_instance = aws.rds.Instance("my-rds",
    allocated_storage=20,
    engine="postgres",
    instance_class="db.t3.micro",
    username="admin",
    password="*****",  # Use Pulumi secrets in real projects!
    db_subnet_group_name=db_subnet_group.name,
    vpc_security_group_ids=[rds_sg.id],
    skip_final_snapshot=True)
```

**What’s going on?**

- The subnet group links our RDS to the VPC subnets.
- The security group opens port 5432 (PostgreSQL’s default) for inbound traffic. (Pro tip: Tighten this to specific IPs on production!)
- The RDS instance is a small `db.t3.micro` PostgreSQL database—perfect for testing.

#### **3. Deploy an EKS Cluster**

Finally, let’s spin up an EKS cluster for running Kubernetes workloads. We’ll need an IAM role first:

```python
import json

# Create an IAM role for EKS
eks_role = aws.iam.Role("eks-role",
    assume_role_policy=json.dumps({
        "Version": "2012-10-17",
        "Statement": [{
            "Action": "sts:AssumeRole",
            "Principal": {"Service": "eks.amazonaws.com"},
            "Effect": "Allow",
        }]
    }))

# Attach the EKS policy to the role
aws.iam.RolePolicyAttachment("eks-policy",
    role=eks_role.name,
    policy_arn="arn:aws:iam::aws:policy/AmazonEKSClusterPolicy")

# Deploy the EKS cluster
eks_cluster = aws.eks.Cluster("my-cluster-name",
    role_arn=eks_role.arn,
    vpc_config=aws.eks.ClusterVpcConfigArgs(
        subnet_ids=[subnet1.id, subnet2.id]))
```

**What’s this doing?**

- We create an IAM role that EKS can use to manage AWS resources, with the right permissions attached.
- The EKS cluster is deployed in our VPC, using the subnets we defined earlier.

#### **Deploy Everything**

Save this code in a `projectscript.py` file (or whatever you named it during setup). Then, from your project directory, run:

```bash
pulumi up
```

Pulumi will show you a preview of what it’s about to do. Hit “yes” to deploy, and watch your infrastructure come to life in AWS!

## 💡 Best Practices for Efficient Infrastructure

- **Use Pulumi Stacks:** Separate your environments (like dev, staging, and prod) with stacks. It’s like having different playlists for different moods—keeps things organized and safe.
- **Secure Sensitive Data:** Don’t hardcode passwords or keys! Use Pulumi’s secrets feature to encrypt them. (Check out [Pulumi’s docs](https://www.pulumi.com/docs/intro/concepts/secrets/) for how.)
- **Keep Costs in Check:** Pick instance sizes that fit your needs—`t3.micro` for testing, beefier ones for production. Bonus: Explore spot instances for savings!
- **Automate Everything:** Hook Pulumi into your CI/CD pipeline (think GitHub Actions or Jenkins) to deploy changes automatically. No more manual tinkering!

## 🌐 Final Thoughts

And there you have it! You’ve just used Pulumi to build a slick AWS infrastructure with a VPC, RDS database, and EKS cluster—all with the power of real code. Pulumi’s approach makes managing cloud resources feel natural, boosts reusability, and fits right into your DevOps workflows.

Ready to level up your cloud game? Dive deeper with these resources:

- [Pulumi Documentation](https://www.pulumi.com/docs/)
- [Pulumi AWS Examples](https://github.com/pulumi/examples)
