# Automate Infrastructure With IaC using Terraform Part 1

Create an IAM user, name it terraform (ensure that the user has only programatic access to your AWS account) and grant this user AdministratorAccess permissions.
Copy the secret access key and access key ID. Save them in a notepad temporarily.
Configure programmatic access from your workstation to connect to AWS using the access keys copied above and a Python SDK (boto3). You must have Python 3.6 or higher on your workstation.

# Best practices

Ensure that every resource is tagged using multiple key-value pairs. You will see this in action as we go along.
Try to write reusable code, avoid hard coding values wherever possible. (For learning purpose, we will start by hard coding, but gradually refactor our work to follow best practices).

# VPC | Subnets | Security Groups

# Let us create a directory structure

Open your Visual Studio Code and:

	Create a folder called PBL
	Create a file in the folder, name it main.tf

Your setup should look like this.

![image](https://user-images.githubusercontent.com/49937302/125180995-282a4380-e233-11eb-8229-ce08b4e409f6.png)


# Provider and VPC resource section

Add AWS as a provider, and a resource to create a VPC in the main.tf file.
Provider block informs Terraform that we intend to build infrastructure within AWS.
Resource block will create a VPC.

	provider "aws" {
			region = "ap-southeast-1"
	}

	resource "aws_vpc" "main" {
		cidr_block                     = "172.16.0.0/16"
		enable_dns_support             = "true"
		enable_dns_hostnames           = "true"
		enable_classiclink             = "false"
		enable_classiclink_dns_support = "false"
	}

The next thing we need to do, is to download necessary plugins for Terraform to work. These plugins are used by providers and provisioners. At this stage, we only have provider in our main.tf file. So, Terraform will just download plugin for AWS provider.
Lets accomplish this with terraform init command as seen in the below demonstration.

navigate to PBL directory

Run "terraform init"

![image](https://user-images.githubusercontent.com/49937302/125181019-5f98f000-e233-11eb-9468-dd556fcec008.png)

Moving on, let us create the only resource we just defined. aws_vpc. But before we do that, we should check to see what terraform intends to create before we tell it to go ahead and create it.

Run terraform plan
Then, if you are happy with changes planned, execute terraform apply

A new file is created terraform.tfstate This is how Terraform keeps itself up to date with the exact state of the infrastructure. It reads this file to know what already exists, what should be added, or destroyed based on the entire terraform code that is being developed.

If you also observed closely, you would realise that another file gets created during planning and apply. But this file gets deleted immediately. terraform.tfstate.lock.info This is what Terraform uses to track, who is running its code against the infrastructure at any point in time. This is very important for teams working on the same Terraform repository at the same time. The lock prevents a user from executing Terraform configuration against the same infrastructure when another user is doing the same - it allows to avoid duplicates and conflicts.

![image](https://user-images.githubusercontent.com/49937302/125181311-27df7780-e236-11eb-93ea-a2cafa8138a1.png)

# Subnets resource section

According to our architectural design, we require 6 subnets:

2 public
2 private for webservers
2 private for data layer
Let us create the first 2 public subnets.

Add below configuration to the main.tf file:

# Create public subnets1
    resource "aws_subnet" "public1" {
    vpc_id                     = aws_vpc.main.id
    cidr_block                 = "172.16.0.0/24"
    map_public_ip_on_launch    = true
    availability_zone          = "ap-southeast-1a"

}

# Create public subnet2
    resource "aws_subnet" "public2" {
    vpc_id                     = aws_vpc.main.id
    cidr_block                 = "172.16.1.0/24"
    map_public_ip_on_launch    = true
    availability_zone          = "ap-southeast-1b"
}

![image](https://user-images.githubusercontent.com/49937302/125181377-b94ee980-e236-11eb-913f-92b7a729c5b3.png)

let us improve our code by refactoring it.

First, destroy the current infrastructure. Since we are still in development, this is totally fine. Otherwise, DO NOT DESTROY an infrastructure that has been deployed to production.

To destroy whatever has been created run terraform destroy command, and type yes after evaluating the plan.

![image](https://user-images.githubusercontent.com/49937302/125181408-059a2980-e237-11eb-92fd-fc5050de4c7f.png)

# Fixing The Problems By Code Refactoring

Fixing Hard Coded Values: We will introduce variables, and remove hard coding.

Starting with the provider block, declare a variable named region, give it a default value, and update the provider section by referring to the declared variable.

Terraform has a functionality that allows us to pull data which exposes information to us. For example, every region has Availability Zones (AZ). Different regions have from 2 to 4 Availability Zones. With over 20 geographic regions and over 70 AZs served by AWS, it is impossible to keep up with the latest information by hard coding the names of AZs. Hence, we will explore the use of Terraform’s Data Sources to fetch information outside of Terraform. In this case, from AWS

Let us fetch Availability zones from AWS, and replace the hard coded value in the subnet’s availability_zone section.

  # Get list of availability zones
        data "aws_availability_zones" "available" {
        state = "available"
        }

To make use of this new data resource, we will need to introduce a count argument in the subnet block: Something like this.

    # Create public subnet1
    resource "aws_subnet" "public" { 
        count                   = 2
        vpc_id                  = aws_vpc.main.id
        cidr_block              = "172.16.1.0/24"
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]

    }
    
 # Let’s make cidr_block dynamic
 
 We will introduce a function cidrsubnet() to make this happen. It accepts 3 parameters. Let us use it first by updating the configuration, then we will explore its internals.

    # Create public subnet1
    resource "aws_subnet" "public" { 
        count                   = 2
        vpc_id                  = aws_vpc.main.id
        cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]

    }
A closer look at cidrsubnet - this function works like an algorithm to dynamically create a subnet CIDR per AZ. Regardless of the number of subnets created, it takes care of the cidr value per subnet.

Its parameters are cidrsubnet(prefix, newbits, netnum)

- The prefix parameter must be given in CIDR notation, same as for VPC.
- The newbits parameter is the number of additional bits with which to extend the prefix. For example, if given a prefix ending with /16 and a newbits value of 8, the resulting subnet address will have length /24
- The netnum parameter is a whole number that can be represented as a binary integer with no more than newbits binary digits, which will be used to populate the additional bits added to the prefix

# The final problem to solve is removing hard coded count value

Now we can simply update the public subnet block like this

# Create public subnet
    resource "aws_subnet" "public" { 
        count                   = length(data.aws_availability_zones.available.names)
        vpc_id                  = aws_vpc.main.id
        cidr_block              = cidrsubnet(var.vpc_cidr, 8 , count.index)
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]

    }

It will return base on all the availabile zone for the region which is not the desire outcom

Now, let us fix this.

Declare a variable to store the desired number of public subnets, and set the default value

	variable "preferred_number_of_public_subnets" {
	  default = 2
	}
Next, update the count argument with a condition. Terraform needs to check first if there is a desired number of subnets. Otherwise, use the data returned by the length function. See how that is presented below.

# Create public subnets
	resource "aws_subnet" "public" {
	  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
	  vpc_id = aws_vpc.main.id
	  cidr_block              = cidrsubnet(var.vpc_cidr, 8 , count.index)
	  map_public_ip_on_launch = true
	  availability_zone       = data.aws_availability_zones.available.names[count.index]

	}

Explanation: 
The first part var.preferred_number_of_public_subnets == null checks if the value of the variable is set to null or has some value defined.

The second part ? and length(data.aws_availability_zones.available.names) means, if the first part is true, then use this. In other words, if preferred number of public subnets is null (Or not known) then set the value to the data returned by lenght function.

The third part : and var.preferred_number_of_public_subnets means, if the first condition is false, i.e preferred number of public subnets is not null then set the value to whatever is definied in var.preferred_number_of_public_subnets

# Introducing variables.tf & terraform.tfvars

Instead of havng a long list of variables in main.tf file, we can actually make our code a lot more readable and better structured by moving out some parts of the configuration content to other files.

We will put all variable declarations in a separate file
And provide non default values to each of them

variables.tf -> contain all declared variables
main.tf -> the main terraform code
terraform.tfvars -> contain non default values of variables

variables.tf
![image](https://user-images.githubusercontent.com/49937302/125185388-283c3a80-e257-11eb-9cdc-f39299c71b2c.png)

terraform.tfvars
![image](https://user-images.githubusercontent.com/49937302/125185401-35592980-e257-11eb-82c2-fbd6dd2ca393.png)

main.tf
![image](https://user-images.githubusercontent.com/49937302/125185379-15c20100-e257-11eb-8467-b5326175887c.png)

verify using terraform plan

![image](https://user-images.githubusercontent.com/49937302/125185433-5e79ba00-e257-11eb-8c5b-c678de5334c1.png)
