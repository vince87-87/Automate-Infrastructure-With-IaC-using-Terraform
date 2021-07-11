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

# Provider and VPC resource section

Add AWS as a provider, and a resource to create a VPC in the main.tf file.
Provider block informs Terraform that we intend to build infrastructure within AWS.
Resource block will create a VPC.

The next thing we need to do, is to download necessary plugins for Terraform to work. These plugins are used by providers and provisioners. At this stage, we only have provider in our main.tf file. So, Terraform will just download plugin for AWS provider.
Lets accomplish this with terraform init command as seen in the below demonstration.
