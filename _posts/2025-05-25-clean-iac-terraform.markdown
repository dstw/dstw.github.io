---
layout: post
title: "Clean Infrastructure as Code with Terraform"
date: 2025-05-24 09:31:39 +0700
comments: true
---

![Header](assets/images/header_clean_iac.png)

Infrastructure as Code (IaC) allows engineering teams to manage cloud resources declaratively using configuration files. But writing Terraform that is not only functional but clean, DRY, and easy to maintain is the difference between a stable system and a future debugging nightmare.

In this post, we’ll explore how using `locals` and `for_each` together in Terraform can significantly improve the readability, maintainability, and scalability of your infrastructure code. We'll also walk through a real-world example and explain why this technique is worth adopting.

---

## What Are `locals` and `for_each` in Terraform?

Before diving into examples, it’s important to understand these two key constructs in Terraform.

### `locals`

The `locals` block in Terraform lets you define named expressions or reusable data that can be referenced throughout your configuration. This is useful when you want to centralize values that are repeated, derive values from other inputs, or simplify your code by abstracting out logic.

Think of `locals` like internal variables. They help you define once and use many times without hardcoding values in multiple places.

### `for_each`

The `for_each` meta-argument allows you to dynamically create multiple instances of a resource or module by iterating over a map or list. It’s particularly useful when you have to provision multiple similar resources — like a fleet of EC2 instances, a group of users, or several security groups.

By combining `locals` and `for_each`, you can write Terraform code that is not only modular but also extensible, with minimal duplication.

---

## Real-World Example: Managing Multiple Security Groups

Consider this real-world scenario: you're building infrastructure for a growing startup. New environments are being spun up for development, QA, staging, and production. Each needs its own set of security groups to control access to services.

Instead of repeating similar blocks of code for each security group, you define all the group configurations in one place using `locals`. Then you use `for_each` to dynamically create the resources.

This approach is scalable, consistent, and easier to maintain — especially as environments grow.

### Step 1: Define Local Configuration

```hcl
locals {
  security_groups = {
    web = {
      name        = "web-sg"
      description = "Allow HTTP and HTTPS"
      ingress = [
        { from_port = 80,  to_port = 80,  protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
        { from_port = 443, to_port = 443, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] }
      ]
    }
    ssh = {
      name        = "ssh-sg"
      description = "Allow SSH"
      ingress = [
        { from_port = 22, to_port = 22, protocol = "tcp", cidr_blocks = ["10.0.0.0/16"] }
      ]
    }
  }
}
```

Here, you define a `security_groups` map that includes each group’s name, description, and ingress rules. This single block gives you a centralized, declarative definition of your access policies.

Adding a new security group? Just add another entry to the map — no need to touch the resource logic.

### Step 2: Create Security Groups with `for_each`

```hcl
resource "aws_security_group" "sg" {
  for_each = local.security_groups

  name        = each.value.name
  description = each.value.description
  vpc_id      = "vpc-abc123" # Replace with your actual VPC ID
}
```

This resource block dynamically creates a security group for each entry in your local map. By iterating with `for_each`, you avoid creating multiple, nearly identical resource blocks.

It also means you get predictable naming and configuration across your environments, without extra code.

### Step 3: Define Ingress Rules Dynamically

```hcl
resource "aws_security_group_rule" "ingress" {
  for_each = {
    for sg_key, sg_value in local.security_groups :
    "${sg_key}" => sg_value
  }

  count = length(each.value.ingress)

  type              = "ingress"
  from_port         = each.value.ingress[count.index].from_port
  to_port           = each.value.ingress[count.index].to_port
  protocol          = each.value.ingress[count.index].protocol
  cidr_blocks       = each.value.ingress[count.index].cidr_blocks
  security_group_id = aws_security_group.sg[each.key].id
}
```

This block uses a loop and count index to apply multiple ingress rules to each group. While more advanced, it’s highly reusable — and allows for any number of rules per group.

---

## Why This Pattern Works

### Less Repetition

When you use `locals` and `for_each`, you don’t have to repeat resource blocks for every variation of a resource. Instead of copy-pasting a block three times with minor changes, you define those changes in a local map.

This eliminates unnecessary duplication and makes your codebase easier to read and review. It also reduces the cognitive overhead when making changes, as you only have to update one place.

### Easier Maintenance

If something changes — like a port number, description, or a CIDR block — you only need to update it in the `locals` block. That means fewer chances of introducing inconsistencies across resources.

Centralization also makes it easier for other team members to understand the configuration. Everything is defined clearly and in one place.

### Better Scalability

As your infrastructure grows, the benefits of this pattern compound. Whether you're provisioning ten or a hundred resources, you don't have to write more code — just update the input data.

This is critical when dealing with multi-environment or multi-region deployments. One `locals` block can serve as the source of truth for many resource definitions.

### Improved Readability

By abstracting the logic into maps and iterating over them, you’re separating **what** needs to be created from **how** it gets created. This leads to cleaner, more declarative code — which is easier to review, audit, and document.

### Fewer Errors

Reducing repetition and centralizing definitions lowers the chance of human error. No more mismatched names or incorrect ports because of a bad copy-paste.

Terraform will also show you exactly what’s changing in the plan, making reviews simpler and more reliable.

---

## Summary

Writing clean Infrastructure as Code is about more than provisioning resources — it’s about building systems that are sustainable and understandable as your team and complexity grow.

By using `locals` and `for_each`, you unlock a simple yet powerful pattern in Terraform: defining your infrastructure **once**, and letting Terraform do the heavy lifting.

This approach results in cleaner, more maintainable, and error-resistant code — which is exactly what you want in production environments.

If you find yourself repeating resource blocks or copy-pasting similar code, take a step back. Try refactoring it using `locals` and `for_each`. Your future self — and your team — will thank you.
