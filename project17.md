# Automate Infrastructure With IaC using Terraform. Part 2

# Networking

# Private subnets & best practices

Create 4 private subnets keeping in mind following principles:

Make sure you use use variables or length() function to determine the number of AZs
Use variables and cidrsubnet() function to allocate vpc_cidr for subnets
Keep variables and resources in separate files for better code structure and readability
Tags all the resources you have created so far. Explore how to use format() and count functions to automatically tag subnets with its respective number.

  tags = merge(
      local.default_tags,
      {
        Name = format("PrivateSubnet-%s", count.index)
      } 
    )

# Internet Gateways & format() function

    resource "aws_internet_gateway" "ig" {
      vpc_id = aws_vpc.main.id
      tags = merge(
        local.default_tags,
        {
          Name = format("%s-%s!", aws_vpc.main.id,"IG")
        } 
      )
    }

*Explaination: Did you notice how we have used format() function to dynamically generate a unique name for this resource? The first part of the %s takes the interpolated value of aws_vpc.main.id while the second %s appends a literal string IG and finally an exclamation mark is added in the end.

If any of the resources being created is either using the count function, or creating multiple resources using a loop, then a key-value pair that needs to be unique must be handled differently.

# NAT Gateways

Create 2 NAT Gateways and 2 Elastic IP (EIP) addresses

create the NAT Gateways in a new file called natgateway.tf.

resource "aws_eip" "nat_eip" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc        = true
  depends_on = [aws_internet_gateway.ig]
  tags = merge(
    local.default_tags,
    {
      Name = format("EIP-%s",var.environment)
    } 
  )
}

  resource "aws_nat_gateway" "nat" {
    count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
    allocation_id = aws_eip.nat_eip[count.index].id
    subnet_id     = element(aws_subnet.public.*.id, 0)
    depends_on    = [aws_internet_gateway.ig]
    tags = merge(
      local.default_tags,
      {
        Name   =  format("Nat-%s",var.environment)
      } 
    )
  }

# AWS Routes

Create a file called route_tables.tf and use it to create routes for both public and private subnets, create the below resources.

    #CUSTOM ROUTE TABLE FOR PUBLIC
    resource "aws_route_table" "RTB-Public" {
      count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets  
      vpc_id = aws_vpc.main.id

    route {
      cidr_block = var.DEFAULT_route_destination
      gateway_id = aws_internet_gateway.ig.id
    }

    tags = {
      Name = format("RTB-PUB-%s",var.environment)
    }
  }

    #CUSTOM ROUTE TABLE FOR Web Private
    resource "aws_route_table" "RTB-WEB-Private" {
      count  = var.preferred_number_of_private_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets
      vpc_id = aws_vpc.main.id

    route {
      cidr_block = var.DEFAULT_route_destination
      gateway_id = aws_nat_gateway.nat[count.index].id
    }

    tags = {
      Name = format("RTB-WEB-PRIVATE-%s",var.environment)
    }
  }

    #CUSTOM ROUTE TABLE FOR Data Private

    resource "aws_route_table" "RTB-Data-Private" {
      count  = var.preferred_number_of_private_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets
      vpc_id = aws_vpc.main.id

    route {
      cidr_block = var.DEFAULT_route_destination
      gateway_id = aws_nat_gateway.nat[count.index].id
    }

    tags = {
      Name = format("RTB-Data-PRIVATE-%s",var.environment)
    }
  }

    #ROUTETABLE ASSOCIATION TO PUBLIC
    resource "aws_route_table_association" "RTB-PUB-Association" {
      count          = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets
      subnet_id      = aws_subnet.public.*.id[count.index]
      route_table_id = aws_route_table.RTB-Public[count.index].id
    }

    #ROUTETABLE ASSOCIATION TO PRIVATE
    resource "aws_route_table_association" "RTB-Private-WEB-Association" {
      count          = var.preferred_number_of_private_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets
      subnet_id      = aws_subnet.WEB-Private.*.id[count.index]
      route_table_id = aws_route_table.RTB-WEB-Private[count.index].id
    }

    #ROUTETABLE ASSOCIATION TO PRIVATE
    resource "aws_route_table_association" "RTB-Private-Data-Association" {
      count          = var.preferred_number_of_private_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets
      subnet_id      = aws_subnet.Data-Private.*.id[count.index]
      route_table_id = aws_route_table.RTB-Data-Private[count.index].id
    }

run terraform plan and apply

![image](https://user-images.githubusercontent.com/49937302/126053782-470905dc-9655-42ea-849a-52049becae5b.png)

A little bit more about Tagging
Tagging is a straightforward, but a very powerful concept that helps you manage your resources much more efficiently:

Resources are much better organized in ‘virtual’ groups

They can be easily filtered and searched from console or programmatically

Billing team can easily generate reports and determine how much each part of infrastructure costs how much (by department, by type, by environment, etc.)

You can easily determine resources that are not being used and take actions accordingly

If there are different teams in the organisation using the same account, tagging can help differentiate who owns which resources.

Note: You can add multiple tags as a default set

![image](https://user-images.githubusercontent.com/49937302/126054322-00c4cdd5-032a-4595-8b77-e999be022b63.png)

![image](https://user-images.githubusercontent.com/49937302/126054330-fecd8946-0889-4676-a19f-4fb8ec65b291.png)

terraform plan show that it the default tags are applied

![image](https://user-images.githubusercontent.com/49937302/126054338-8f48d1e7-164a-4616-a6ce-373efac7817a.png)

# AWS Identity and Access Management
# IaM and Roles

Create AssumeRole

Assume Role uses Security Token Service (STS) API that returns a set of temporary security credentials that you can use to access AWS resources that you might not normally have access to. These temporary credentials consist of an access key ID, a secret access key, and a security token. Typically, you use AssumeRole within your account or for cross-account access.

Add the following code to a new file named security.tf

     resource "aws_iam_role" "ec2_instance_role" {
        name = "ec2_instance_role"
        assume_role_policy = <<EOF
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                "Action": "sts:AssumeRole",
                "Principal": {
                    "Service": "ec2.amazonaws.com"
                },
                "Effect": "Allow",
                "Sid": ""
                }
            ]
        }
        EOF
        tags = {
            Name = "aws assume role"
            Environment = var.environment
            }
        }

Attach the Policy to the IAM Role
This is where, we will be attaching the policy which we created above, to the role we created in the first step.

    resource "aws_iam_role_policy_attachment" "test-attach" {
        role       = aws_iam_role.ec2_instance_role.name
        policy_arn = aws_iam_policy.policy.arn
    }
    
Create an Instance Profile and interpolate the IAM Role

    resource "aws_iam_instance_profile" "ip" {
            name = "aws_instance_profile_test"
            role =  aws_iam_role.ec2_instance_role.name
        }
# Compute and Security
# Resources to be created inside public subnets

Upload SSH keys to EC2 key pairs

Create 2 Bastion EC2 Instances along with Security Groups

Create an Application Load Balancer (ALB)

Create an Auto Scaling Group (ASG)

# Upload SSH key to EC2 key pairs

