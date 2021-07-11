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
