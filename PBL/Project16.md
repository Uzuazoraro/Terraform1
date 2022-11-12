## AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM PART 1

### Prerequisites before you begin writing Terraform code

=== Create an IAM user, name it terraform
IAM Dashboard > users > add user> click Next:permissions > click create group (select AdministratorAccess) > 

![creation of terraform user](./terraform user being created.png)
![creation of terraform user1](./terraform user being created1.png)
![creation of terreform user](./terraform user created.png)

### Create an S3 bucket to store Terraform state file.

Under AWS services > Click S3 > Create buckets > Name (Micolo-dev-terraform-bucket) > Bucket Versioning (Enable) > Tags (key: Name, Valus:Micolo-dev-terraform-bucket) > Server side encryption (Disable) > Create bucket 

![creation of bucket](./bucket created.png)

### VPC | SUBNETS | SECURITY GROUPS

Create a folder called PBL
Create a file in the folder, name it main.tf

### Provider and VPC resource section

Set up Terraform CLI as per this instruction.

Add AWS as a provider, and a resource to create a VPC in the main.tf file.
Provider block informs Terraform that we intend to build infrastructure within AWS.
Resource block will create a VPC.
provider "aws" {
  region = "us-east-1"
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = "172.16.0.0/16"
  enable_dns_support             = "true"
  enable_dns_hostnames           = "true"
  enable_classiclink             = "false"
  enable_classiclink_dns_support = "false"
}

### Initialize Terraform with this command:

`terraform init`

![initialization of terraform](./terraform installed and initialized.png)

### `terreform plan`

![terraform plan output](./terraform plan.png)

`terraform validate`

![validation of terraform](./terraform validate.png)

![confirmation of terraform plan](./terraform apply.png)

![confirmation of subnets creation](./subnets created.png)

### Improve Code by Refactoring

=== First, you've to destroy before refactoring code. Use the command below:

`terraform destroy`
![confirming terraform destroy](./terraform destroy.png)
![confirmation of destroying terraform](./terraform destroyed.png)
![confirmation of destroying terraform](./created subnets removed by terraform destroyed.png)

Fixing The Problems By Code Refactoring.
=======================================

Starting with the provider block, declare a variable named region, give it a default value, and update the provider section by referring to the declared variable.
    variable "region" {
        default = "eu-central-1"
    }

    provider "aws" {
        region = var.region
    }
Do the same to cidr value in the vpc block, and all the other arguments.
    variable "region" {
        default = "eu-central-1"
    }

    variable "vpc_cidr" {
        default = "172.16.0.0/16"
    }

    variable "enable_dns_support" {
        default = "true"
    }

    variable "enable_dns_hostnames" {
        default ="true" 
    }

    variable "enable_classiclink" {
        default = "false"
    }

    variable "enable_classiclink_dns_support" {
        default = "false"
    }

    provider "aws" {
    region = var.region
    }

    # Create VPC
    resource "aws_vpc" "main" {
    cidr_block                     = var.vpc_cidr
    enable_dns_support             = var.enable_dns_support 
    enable_dns_hostnames           = var.enable_dns_support
    enable_classiclink             = var.enable_classiclink
    enable_classiclink_dns_support = var.enable_classiclink

    }


Fixing multiple resource blocks.
================================


Let’s make cidr_block dynamic.
We will introduce a function cidrsubnet() to make this happen. It accepts 3 parameters. Let us use it first by updating the configuration, then we will explore its internals.

    # Create public subnet1
    resource "aws_subnet" "public" { 
        count                   = 2
        vpc_id                  = aws_vpc.main.id
        cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]

    }
A closer look at cidrsubnet – this function works like an algorithm to dynamically create a subnet CIDR per AZ. Regardless of the number of subnets created, it takes care of the cidr value per subnet.

Its parameters are cidrsubnet(prefix, newbits, netnum)

The prefix parameter must be given in CIDR notation, same as for VPC.
The newbits parameter is the number of additional bits with which to extend the prefix. For example, if given a prefix ending with /16 and a newbits value of 4, the resulting subnet address will have length /20
The netnum parameter is a whole number that can be represented as a binary integer with no more than newbits binary digits, which will be used to populate the additional bits added to the prefix
You can experiment how this works by entering the terraform console and keep changing the figures to see the output.

On the terminal, run terraform console
type cidrsubnet("172.16.0.0/16", 4, 0)
Hit enter
See the output
Keep change the numbers and see what happens.
To get out of the console, type exit

![how cidr subnet works](./CIDR subnet.png)


Remove hard coded count value
=============================

Introduce the lenght function.

length(["eu-central-1a", "eu-central-1b", "eu-central-1c"])

This will return a value of 3 when entered in the terraform console.

![testing vpc length in terraform console](./vpc length in terraform console.png)

Update public subnet
====================

# Create public subnet1
    resource "aws_subnet" "public" { 
        count                   = length(data.aws_availability_zones.available.names)
        vpc_id                  = aws_vpc.main.id
        cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]

    }

=== Next, update the count argument with a condition. Terraform needs to check first if there is a desired number of subnets. Otherwise, use the data returned by the lenght function. See how that is presented below.

# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]

}

Now lets break it down:

= The first part var.preferred_number_of_public_subnets == null checks if the value of the variable is set to null or has some value defined.
= The second part ? and length(data.aws_availability_zones.available.names) means, if the first part is true, then use this. In other words, if preferred number of public subnets is null (Or not known) then set the value to the data returned by lenght function.
= The third part : and var.preferred_number_of_public_subnets means, if the first condition is false, i.e preferred number of public subnets is not null then set the value to whatever is definied in var.preferred_number_of_public_subnets

![terraform plan](./updated terraform plan.png)

### INTRODUCING VARIABLES.TF & AMP; TERRAFORM.TFVARS

= We will put all variable declarations in a separate file
= And provide non default values to each of them

1. Create a new file and name it variables.tf
2. Copy all the variable declarations into the new file.
3. Create another file, name it terraform.tfvars
4. Set values for each of the variables.

![terraform format](./terraform fmt.png)

=== Use `terraform apply --auto-approve` to apply the terraform plan.
![apply terraform plan](./terraform apply.png)
![apply 
terraform plan](./terraform apply continued.png)