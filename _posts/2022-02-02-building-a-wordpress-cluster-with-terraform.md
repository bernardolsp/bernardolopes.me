---
layout: post
title: Building a WordPress Cluster with Terraform
date: 2022-02-02 19:21 -0300
---

### **Building a WordPress Cluster - Part 01 (Networking)**
using Terraform to deploy:
- an RDS database; 
- an EKS Cluster;
- an Amazon EFS file system
- a VPC
- a Route Table
- an Internet Gateway
- [...] perhaps a bit more.  

#### **Configuring the local environment** 

We'll be using the *Layer Style Guide*, but perhaps a bit reduced, due to the scope of this project.
This will be our starting point:

```
➜  WordPress-RDS-EKS-Cluster tree                                
.
├── k8s
└── terraform
    └── layers
        ├── computing
        ├── efs
        └── networking
```

---

I'd like to start with the basics - **networking**. As we'll use a managed database the setup will be a tad simpler. However, in this guide we'll use **Invalid CIDRs** for our pod network. One can see a few details on how it works by heading to [this guide](https://medium.com/webstep/dont-let-your-eks-clusters-eat-up-all-your-ip-addresses-1519614e9daa).

Other than the enviroment variables correctly set (so we can run terraform plan and terraform apply), there will be no shortcuts.

In our network, we'll need:
- A VPC
- At least one subnet (but we'll use two).

Create **four files**, to begin with on networking folder - *provider.tf, subnet.tf, vpc.tf, variables.tf*. That way, we can keep everything in its place.

Content of **provider.tf**:
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.74.0" # Current Latest
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```
Then, run `terraform init` from your terminal.

We'll use a /16 for our main subnet, so as for our secondary (invalid) one.

This is the end result of the **vpc.tf** file:

```hcl
resource "aws_vpc" "main_vpc" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name      = "default-vpc",
    ManagedBy = "Terraform"
  }
}

resource "aws_vpc_ipv4_cidr_block_association" "invalid_cidr" {
  vpc_id     = aws_vpc.main_vpc.id
  cidr_block = "100.64.0.0/16"
}
```

This is mainly what we've been wondering about - a VPC with a big CIDR and an ipv4 cidr association to handle our secondary block, full of invalid IPs for our Pods.

Now, we need to define which subnet is assigned to which AZs. There are several ways to do it - I'll assign them in **variables.tf**, using a list of objects. This is what makes the most sense to me, to reduce *code-repeat* but **ymmv**. Let's take a look:

```hcl
variable "valid_azs" {
  default = [
    { "az" : "us-east-1a", "ip" : "10.0.0.0/20" },
    { "az" : "us-east-1b", "ip" : "10.0.16.0/20" },
  { "az" : "us-east-1c", "ip" : "10.0.32.0/20" }]
}

variable "invalid_azs" {
  default = [
    { "az" : "us-east-1a", "ip" : "100.64.0.0/19" },
    { "az" : "us-east-1b", "ip" : "100.64.32.0/19" },
  { "az" : "us-east-1c", "ip" : "100.64.64.0/19" }]
}
```

We can (and will) loop on that.

Let's take a look at the subnets.

```hcl
resource "aws_subnet" "valid-this" {
  for_each          = { for index, az in var.valid_azs : index => az }
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = each.value.ip
  availability_zone = each.value.az
  tags = {
    Name      = "eks-in-${each.value.az}",
    ManagedBy = "Terraform"
  }
}


resource "aws_subnet" "invalid-this" {
  for_each          = { for index, az in var.invalid_azs : index => az }
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = each.value.ip
  availability_zone = each.value.az
  tags = {
    Name      = "invalid-in-${each.value.az}",
    ManagedBy = "Terraform"
  }
  depends_on = [
    aws_vpc_ipv4_cidr_block_association.invalid_cidr
  ]

}
```

The only catch here is the **for_each** loop - it breaks down to a few steps:
- we grab the index on the main list
- with this index in hand, we can break the object in a key/value (that is, for each object in the list, break it in a key/value model"
- we use the values

We also need to explicitly tell Terraform to *wait* for the secondary CIDR to be created in the VPC (as, of course, we're creating subnets with the IPs from such block).

Once we have these, let's create two new files, so our directory resembles this:
```
➜  networking git:(main) ✗ tree                  
.
├── internetgateway.tf
├── provider.tf
├── routetables.tf
├── subnet.tf
├── variables.tf
└── vpc.tf
```

the **internetgateway.tf** is the easiest one - we'll simply allow our VPC to access the internet. This is the snippet:

```hcl
resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "default-eks"
  }
}
```

The **routetables.tf** is also quite simple:

```hcl
resource "aws_route_table" "this" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }
}

resource "aws_route_table_association" "invalid" {
  subnet_id      = aws_subnet.invalid-this[count.index].id
  route_table_id = aws_route_table.this.id
  count = 3
}
resource "aws_route_table_association" "valid" {
  subnet_id      = aws_subnet.valid-this[count.index].id
  route_table_id = aws_route_table.this.id
  count = 3
}
```

A **Route Table** defines how our EC2 instances can communicate with the internet (somewhat like a router-firewall). In this case, we're being quite broad - and allowing all communication (look at the 0.0.0.0).

The **Route Table Association** is associating the subnets with the current route table. We're using *count* to perform a loop in every subnet we're creating.

If everything is right - you can now run *terraform fmt -recursive* to format your code and run *terraform plan*.

The result should be:
`Plan: 16 to add, 0 to change, 0 to destroy.`

Thanks for tuning in so far. You can find this code in [my github repo](https://github.com/bernardolsp/terraform-wordpress-eks)