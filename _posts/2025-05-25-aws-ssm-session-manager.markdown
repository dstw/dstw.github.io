---
layout: post
title: "Securing Bastion-less Access to AWS EC2 Using AWS SSM Session Manager"
date: 2025-05-25 10:33:09 +0700
comments: true
---

![AWS SSM Session Manager](/assets/images/aws_ssm_session_manager.png)

## Introduction

For years, the industry-standard approach to securely access AWS EC2 instances has involved deploying jump hosts (also known as bastion hosts). These are intermediary servers that act as controlled gateways to your private instances, enabling SSH access while minimizing exposure to the public internet.
However, managing jump hosts comes with operational overhead and security risks:

* You must maintain and patch additional servers.
* Managing SSH keys and users can become cumbersome.
* Bastion hosts themselves represent an additional attack surface.
* Open inbound SSH ports increase exposure.

Enter **AWS Systems Manager (SSM) Session Manager** — a powerful tool that enables secure, auditable, portless access to your EC2 instances without needing a single bastion host.
This post explores how and why SSM Session Manager can replace traditional jump hosts, including practical examples, architecture diagrams, and setup instructions.

---

## What is AWS SSM Session Manager?

AWS Systems Manager Session Manager is a fully managed AWS service that allows you to start secure shell (SSH-like) sessions to your EC2 instances **without** opening inbound ports or requiring SSH keys. Instead, connections are routed securely through the AWS API.

Key features include:

* **No open inbound ports:** You don’t need to open port 22 for SSH.
* **IAM-based access control:** Access is controlled via AWS IAM permissions.
* **Auditability:** Session logs can be stored centrally in CloudWatch Logs or S3.
* **Encryption:** All communications are encrypted using TLS.
* **Cross-platform:** Works with both Linux and Windows instances.

---

Let’s start with a story.

At a fast-growing SaaS company, a DevOps team manages dozens of EC2 instances across multiple VPCs. Initially, they used bastion hosts for access — a secure jump server per environment. But maintaining them became a full-time job:

* Open SSH ports drew the attention of scanners.
* SSH key management was a mess.
* Logging and auditing were fragmented.
* Engineers dreaded the extra hops and manual effort.

Eventually, they discovered AWS SSM Session Manager. And everything changed.

### 1. Reduced Attack Surface

With SSM, there's no need to expose port 22. EC2 instances no longer require public IPs or bastion-accessible security groups. You essentially eliminate all traditional SSH entry points.

**Security wins:**

* No open ports
* No need for public subnet
* Encrypted channel by default

### 2. Simplified Access Management

Access is managed through **IAM roles and policies**, not local Linux accounts or SSH keys. You can:

* Grant access via IAM groups
* Revoke access instantly by removing IAM permissions
* Control actions through fine-grained policies

### 3. Operational Efficiency

Say goodbye to provisioning, patching, securing, and monitoring bastion hosts. Everything is handled through the SSM agent, which runs as a background daemon on your EC2 instances.

### 4. Full Session Auditing

You can log session activity (including commands run and duration) to **CloudWatch Logs** or **S3**, helping meet compliance and audit needs.

---

## Architecture Diagram

```
User Workstation
     |
     | IAM Auth
     |
AWS Console / AWS CLI
     |
     | SSM Session
     v
+--------------------+         +----------------------+
|   SSM Service      |<------->|    EC2 Instances     |
| (Mgmt Plane, Logs) |         |  (SSM Agent enabled) |
+--------------------+         +----------------------+
```
---

## How to Set Up SSM Session Manager

### Step 1: Ensure the SSM Agent is Installed and Running

Most recent Amazon Linux, Ubuntu, and Windows AMIs come with the SSM agent preinstalled. If not, install it manually:

```bash
# For Amazon Linux
sudo yum install -y amazon-ssm-agent
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
```

### Step 2: Attach an IAM Role with SSM Permissions to Your Instance

Create or update an IAM role with the following managed policies attached:

* `AmazonSSMManagedInstanceCore`

Attach this IAM role to your EC2 instance.

### Step 3: Verify Instance is Managed

In the AWS Console under Systems Manager > Managed Instances, verify your instance appears and is “Online.”

### Step 4: Start a Session

You can start a session in multiple ways:

* **AWS CLI:**

```bash
aws ssm start-session --target instance-id
```

* **AWS Console:**

Go to EC2 > Instances, select your instance, then click **Connect** > **Session Manager** tab.

---

## Optional: Log Sessions to CloudWatch or S3

Enable centralized auditing by configuring SSM to send logs to a CloudWatch Log Group or S3 bucket. You can do this by:

1. Creating a CloudWatch Log Group (optional)
2. Setting up an SSM Document (e.g. `AWS-StartInteractiveCommand` with logging enabled)
3. Using `AWS-SSM-SessionManagerRunShell` with logging configuration

---

## Automating Setup with Terraform

To simplify infrastructure setup and enforce consistency, you can use Terraform to provision the IAM role and EC2 instance with the right SSM permissions.

### Terraform IAM Role and Policy for SSM

```
resource "aws_iam_role" "ssm_role" {
  name = "ec2_ssm_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect = "Allow",
      Principal = {
        Service = "ec2.amazonaws.com"
      },
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ssm_managed_policy" {
  role       = aws_iam_role.ssm_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}
```

### Attaching the Role to EC2 Instance

```
resource "aws_instance" "example" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"
  iam_instance_profile = aws_iam_instance_profile.ssm_profile.name

  # Other instance configuration ...
}

resource "aws_iam_instance_profile" "ssm_profile" {
  name = "ssm_instance_profile"
  role = aws_iam_role.ssm_role.name
}
```

This Terraform snippet will create an IAM role with the necessary SSM permissions, attach it to an instance profile, and launch the EC2 instance with that profile. This enables SSM Session Manager connectivity out of the box.

---

## Example of Fine-Grained IAM Policies for Session Manager Access

In a production environment, you likely want to restrict who can start sessions, and possibly limit access to specific instances.
Here is an example IAM policy granting session start permission on a subset of instances by tag:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:StartSession",
        "ssm:DescribeSessions",
        "ssm:GetSession",
        "ssm:TerminateSession"
      ],
      "Resource": [
        "arn:aws:ssm:*:*:session/${aws:username}*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:StartSession"
      ],
      "Resource": [
        "arn:aws:ec2:*:*:instance/*"
      ],
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/Environment": "Production"
        }
      }
    }
  ]
}
```

This policy:

* Allows starting SSM sessions only to EC2 instances tagged `Environment=Production`.
* Allows managing sessions owned by the user.

---

## Best Practices for Bastion-less Access

* **Enforce least privilege with IAM policies:** Only grant SSM session permissions to necessary users or groups.
* **Enable session logging:** Configure CloudWatch Logs or S3 bucket to store session activity for audits.
* **Use Multi-Factor Authentication (MFA):** Protect AWS accounts and roles with MFA for enhanced security.
* **Combine with AWS Organizations SCPs:** For enterprise control over session access.
* **Regularly review permissions:** Continuously monitor IAM policies and session logs.

---

## Audit and Compliance Benefits

With Session Manager:

* All sessions can be recorded and reviewed.
* You can track exactly who accessed what and when.
* Logs can be integrated into SIEM tools for real-time monitoring.

This centralized audit trail helps you maintain compliance with standards such as PCI DSS, HIPAA, and SOC 2.

---

## Frequently Asked Questions

**Q: Can I use SSM Session Manager with EC2 instances in private subnets?**  
A: Yes — as long as they can reach the SSM endpoint via NAT or VPC endpoint.

**Q: Does this mean I should never use SSH again?**  
A: Not necessarily. SSH still has use cases (e.g., SCP file transfers, troubleshooting), but SSM covers most day-to-day operations.

**Q: Is this HIPAA/GDPR compliant?**  
A: With session logging and IAM control, SSM helps support many compliance frameworks. Always consult your auditor for confirmation.

---

## Conclusion

Jump hosts have served us well, but in a cloud-native environment, they add unnecessary complexity and risk. AWS SSM Session Manager is a secure, cost-effective, and scalable replacement that streamlines access to EC2 instances.
By adopting bastion-less access, you reduce operational overhead, improve your security posture, and gain full control and visibility into remote sessions.
If you’re still using jump hosts or managing SSH keys and open ports, now is the time to consider switching to AWS Systems Manager Session Manager.

---

**#AWS #Security #CloudSecurity #DevOps #30DaysOfDevOps**
