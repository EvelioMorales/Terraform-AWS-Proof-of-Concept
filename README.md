# Terraform AWS Proof of Concept

Same as [AWS Proof of Concept](https://github.com/EvelioMorales/AWS-Proof-of-Concept/tree/main) this project will consist of basic infrastructure on AWS to host a Proof of Concept environment. The architecture will include both public and private subnets and span multiple Availability Zones to test failover and disaster recovery scenarios. This project will host Internet-facing applications and other applications that need to access the Internet to retrive security and operating system updates. The difrence will be that this project will be done in *__'Terraform'__*.

![Diagram]()

# 1. Create a Folder 

I will start by creating a __New Folder__ and naming it *__'AWS_PoC_Terrafrom'__*.

![Name Folder]()

# 2. Open Folder in VSCode and create files 

In VSCode I will open *__'AWS_PoC_Terrafrom'__* folder. 

![open folder]()

Next I will createa 2 files one I will name *__main.tf__* and the other *__variables.tf__*.

![files created]()

# 3. Add code to files

In the *__variables.tf__* file I will add the following:

```terraform
variable "aws_region" {
type = string
default = "us-east-1"
}
variable "vpc_name" {
type = string
default = "demo_vpc"
}
variable "vpc_cidr" {
type = string
default = "10.0.0.0/16"
}
variable "private_subnets" {
default = {
"private_subnet_1" = 1
"private_subnet_2" = 2
"private_subnet_3" = 3
}
}
variable "public_subnets" {
default = {
"public_subnet_1" = 1
"public_subnet_2" = 2
"public_subnet_3" = 3
}
}
```
In *__main.tf__* file I will add:

```terraform
# Configure the AWS Provider
provider "aws" {
region = "us-east-1"
}
#Retrieve the list of AZs in the current AWS region
data "aws_availability_zones" "available" {}
data "aws_region" "current" {}
#Define the VPC
resource "aws_vpc" "vpc" {
cidr_block = var.vpc_cidr
tags = {
Name = var.vpc_name
Environment = "demo_environment"
Terraform = "true"
}
}
#Deploy the private subnets
resource "aws_subnet" "private_subnets" {
for_each = var.private_subnets
vpc_id = aws_vpc.vpc.id
cidr_block = cidrsubnet(var.vpc_cidr, 8, each.value)
availability_zone = tolist(data.aws_availability_zones.available.names)[
each.value]
tags = {
Name = each.key
Terraform = "true"
}
}
#Deploy the public subnets
resource "aws_subnet" "public_subnets" {
for_each = var.public_subnets
vpc_id = aws_vpc.vpc.id
cidr_block = cidrsubnet(var.vpc_cidr, 8, each.value + 100)
availability_zone = tolist(data.aws_availability_zones.available.
names)[each.value]
map_public_ip_on_launch = true
tags = {
Name = each.key
Terraform = "true"
}
}
#Create route tables for public and private subnets
resource "aws_route_table" "public_route_table" {
vpc_id = aws_vpc.vpc.id
route {
cidr_block = "0.0.0.0/0"
gateway_id = aws_internet_gateway.internet_gateway.id
#nat_gateway_id = aws_nat_gateway.nat_gateway.id
}
tags = {
Name = "demo_public_rtb"
Terraform = "true"
}
}
resource "aws_route_table" "private_route_table" {
vpc_id = aws_vpc.vpc.id
route {
cidr_block = "0.0.0.0/0"
# gateway_id = aws_internet_gateway.internet_gateway.id
nat_gateway_id = aws_nat_gateway.nat_gateway.id
}
tags = {
Name = "demo_private_rtb"
Terraform = "true"
}
}
#Create route table associations
resource "aws_route_table_association" "public" {
depends_on = [aws_subnet.public_subnets]
route_table_id = aws_route_table.public_route_table.id
for_each = aws_subnet.public_subnets
subnet_id = each.value.id
}
resource "aws_route_table_association" "private" {
depends_on = [aws_subnet.private_subnets]
route_table_id = aws_route_table.private_route_table.id
for_each = aws_subnet.private_subnets
subnet_id = each.value.id
}
#Create Internet Gateway
resource "aws_internet_gateway" "internet_gateway" {
vpc_id = aws_vpc.vpc.id
tags = {
Name = "demo_igw"
}
}
#Create EIP for NAT Gateway
resource "aws_eip" "nat_gateway_eip" {
domain = "vpc"
depends_on = [aws_internet_gateway.internet_gateway]
tags = {
Name = "demo_igw_eip"
}
}
#Create NAT Gateway
resource "aws_nat_gateway" "nat_gateway" {
depends_on = [aws_subnet.public_subnets]
allocation_id = aws_eip.nat_gateway_eip.id
subnet_id = aws_subnet.public_subnets["public_subnet_1"].id
tags = {
Name = "demo_nat_gateway"
}
}
```
# 4. Connect and Deploy 

Onece the code has been added using trhe terminal I will connect to AWS by running the following commands

```bash
export AWS_ACCESS_KEY_ID="<ADD ACCESS KEY>"
export AWS_SECRET_ACCESS_KEY="<ADD SECRET KEY>"
```
onece connected to AWS the first step to using Terraform is initializing the working directory. In your bash session, type the

```bash
terraform init
```
Now that our working directory is initialized, we can create a plan for execution. This will provide a
preview of the changes to our AWS environment. To create a plan, execute the following command:

```bash
terraform plan
```
Bash will return information on all the resources that will be created. For our final step to create ourAWS resources,we need to apply the configuration. An apply will instruct
Terraform to create the resources in AWS that are defined in our configuration file(s). And as we saw in
our plan, it will create 18 resources for us. To execute the Terraform, run the following command:

```bash
terraform apply -auto-approve
```

Onece the apply command has been ran a list of all the resources that where created will show followed by an Apply complete! and the number of recsources added. Now to verify I can log in to AWS console to view the resources created. Finally I will run __terraform destroy__ to delete all the resources created 

```bash
terraform destroy -auto-approve
```

