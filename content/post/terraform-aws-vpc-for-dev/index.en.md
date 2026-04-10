---
title: "Terraform-Based Cost Optimization for NAT Gateway in the Development Environment"
description: A cost-optimization strategy that reduces NAT Gateway expenses in the development environment by conditionally creating and removing NAT-related resources with Terraform
slug: terraform-based-cost-optimization-for-nat-gateway-in-the-development-environment
date: 2026-04-10T16:17:01+09:00
image: image.png
math: 
license: 
hidden: false
comments: true
draft: false

tags:
    - AWS
    - Terraform

categories:
    - AWS
    - Terraform
---

## 💰 Introduction

In our college capstone design project, our budget is too limited to keep paying ongoing AWS cloud costs. \
However, our infrastructrue does not need to run 24/7. \
Since the NAT Gateway incurs relatively high costs simply by begin provisioned, we decided to remove it during non-working hours. \
This allows us to reduce unncessary expenses while keeping the envionment available when the team is actively working. 

## 💡 Strategy

![Terrafrom Strategy](image.png)

The strategy is to control NAT Gateway provisioning through the `natgw_azs` variable. \
If `natgw_azs`, which is of type `list(string)`, is empty, Terraform will not create any NAT Gateway or related route table entries. \
If those resources already exist, Terraform will remove them during apply. \
Based on this behavior, we can build a module that conditionally manages NAT Gateways and routing rules, then integrate it with automation tools to enable costs optimization during non-working hours.


## 🖥️ Terraform Code

### Module

This module creates Regional NAT Gateway when variable `natgw_azs` has any elements(`availability zones`). \
If `natgw_azs` is empty, NAT Gateway is desired to be destroyed.

```hcl
# main.tf
locals {
  nat_enabled = length(var.natgw_azs) > 0
}

#########
## VPC ##
#########
resource "aws_vpc" "vpc" {
  cidr_block           = var.cidr_block
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.cluster_name}-vpc"
  }
}

#############
## Subnets ##
#############
resource "aws_subnet" "public" {
  for_each = var.public_subnets

  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = each.value.cidr
  availability_zone       = var.azs[each.value.az]
  map_public_ip_on_launch = false

  tags = {
    Name                                        = "${var.cluster_name}-public-subnet-${each.key}"
    "kubernetes.io/role/elb"                    = "1"
  }
}

resource "aws_subnet" "private" {
  for_each = var.private_subnets

  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = each.value.cidr
  availability_zone       = var.azs[each.value.az]
  map_public_ip_on_launch = false

  tags = {
    Name                                        = "${var.cluster_name}-private-subnet-${each.key}"
    "kubernetes.io/role/internal-elb"           = "1"
  }
}

resource "aws_subnet" "db" {
  for_each = var.db_subnets

  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = each.value.cidr
  availability_zone       = var.azs[each.value.az]
  map_public_ip_on_launch = false

  tags = {
    Name = "${var.cluster_name}-db-subnet-${each.key}"
  }
}

#################
## IGW & NATGW ##
#################
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.vpc.id

  tags = {
    Name = "${var.cluster_name}-igw"
  }
}

# Create EIPs for each azs
resource "aws_eip" "regional_nat" {
  for_each = local.nat_enabled ? toset([
    for az in var.natgw_azs : var.azs[az]
  ]) : []

  domain = "vpc"

  tags = {
    Name = "${var.cluster_name}-regional-nat-${each.key}"
  }
}

# NAT Gateway: created when len(natgw_azs) > 0
resource "aws_nat_gateway" "regional_natgw" {
  count = local.nat_enabled ? 1 : 0

  vpc_id            = aws_vpc.vpc.id
  availability_mode = "regional"

  # Associate EIP alloc & NATGW
  dynamic "availability_zone_address" {
    for_each = aws_eip.regional_nat

    content {
      availability_zone = availability_zone_address.key
      allocation_ids    = [availability_zone_address.value.id]
    }
  }

  tags = {
    Name = "${var.cluster_name}-regional-natgw"
  }
}

###############################
## Route Table & Association ##
###############################
resource "aws_route_table" "public_rtb" {
  vpc_id = aws_vpc.vpc.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = {
    Name = "${var.cluster_name}-public-rtb"
  }
}

resource "aws_route_table" "private_rtb" {
  vpc_id = aws_vpc.vpc.id

  # Create route entry for NATGW if presents
  # aws_nat_gateway.regional_nat returns array, because of `count`
  dynamic "route" {
    for_each = local.nat_enabled ? [1] : []

    content {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.regional_natgw[0].id
    }
  }

  tags = {
    Name = "${var.cluster_name}-private-rtb"
  }
}

resource "aws_route_table" "db_rtb" {
  vpc_id = aws_vpc.vpc.id

  tags = {
    Name = "${var.cluster_name}-db-rtb"
  }
}

resource "aws_route_table_association" "public_rtb" {
  for_each       = aws_subnet.public
  subnet_id      = each.value.id
  route_table_id = aws_route_table.public_rtb.id
}

resource "aws_route_table_association" "private_rtb" {
  for_each       = aws_subnet.private
  subnet_id      = each.value.id
  route_table_id = aws_route_table.private_rtb.id
}

resource "aws_route_table_association" "db_rtb" {
  for_each       = aws_subnet.db
  subnet_id      = each.value.id
  route_table_id = aws_route_table.db_rtb.id
}
```

```hcl
# variables.tf
variable "cluster_name" {
  description = "Kubernetes cluster name for prefix"
  type        = string
}

variable "azs" {
  description = "Availability zone aliases"
  type        = map(string)
}

variable "cidr_block" {
  description = "CIDR block for vpc."
  type        = string
}

variable "public_subnets" {
  description = "Public subnets"
  type = map(object({
    cidr = string
    az   = string
  }))
}

variable "private_subnets" {
  description = "Private subnets"
  type = map(object({
    cidr = string
    az   = string
  }))
}

variable "db_subnets" {
  description = "DB subnets"
  type = map(object({
    cidr = string
    az   = string
  }))
}

variable "natgw_azs" {
  description = "EIP alloc az for NATGW"
  type        = list(string)
}
```

### Usage

Here's the example for using modules. \
You can simply toggle NAT Gateway by changing the argument: `natgw_azs`.

```hcl
# main.tf

#######
# VPC #
#######
module "vpc" {
  source       = "../../../modules/vpc"
  cluster_name = var.cluster_name
  cidr_block   = local.vpc_cidr
  azs = {
    a = "ap-northeast-2a"
    c = "ap-northeast-2c"
  }
  public_subnets = {
    a = {
      cidr = "10.1.0.0/24"
      az   = "a"
    }
    c = {
      cidr = "10.1.1.0/24"
      az   = "c"
    }
  }
  private_subnets = {
    a = {
      cidr = "10.1.100.0/24"
      az   = "a"
    }
    c = {
      cidr = "10.1.101.0/24"
      az   = "c"
    }
  }
  db_subnets = {
    a = {
      cidr = "10.1.200.0/24"
      az   = "a"
    }
    c = {
      cidr = "10.1.201.0/24"
      az   = "c"
    }
  }
  # When reducing the number of NAT gateway AZs,
  # it is safer to first set `natgw_azs` to an empty list and apply,
  # then recreate the NAT gateway with the desired AZs.
  natgw_azs = ["a", "c"]
}

```

## 📚 References

- [Terraform Registry - AWS NAT Gateway](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/nat_gateway)
- [Module Source Code (GitHub)](https://github.com/Hallym-Workerbees/hivewiki-infra/blob/main/modules/vpc/main.tf)
