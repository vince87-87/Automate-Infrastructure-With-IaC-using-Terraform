# Automate Infrastructure With IaC using Terraform. Part 3 - Refactoring

# Configure S3 as backend
    
    resource "aws_s3_bucket" "terraform_state" {
      bucket = "myterraform-vbackend"
      # Enable versioning so we can see the full revision history of our state files
      versioning {
        enabled = true
      }
      # Enable server-side encryption by default
      server_side_encryption_configuration {
        rule {
          apply_server_side_encryption_by_default {
            sse_algorithm = "AES256"
          }
        }
      }
    }
    
    # Configure dynamodb for locking and consistency checking
    resource "aws_dynamodb_table" "terraform_locks" {
      name         = "terraform-locks"
      billing_mode = "PAY_PER_REQUEST"
      hash_key     = "LockID"
      attribute {
        name = "LockID"
        type = "S"
      }
    }

Perform terraform apply to create the s3 and dynamodb resource

perform terraform init
    
    terraform {
      backend "s3" {
        bucket         = "myterraform-vbackend"
        key            = "global/s3/terraform.tfstate"
        region         = "ap-southeast-1"
        dynamodb_table = "terraform-locks"
        encrypt        = true
      }
    }
 
![image](https://user-images.githubusercontent.com/49937302/126870916-92a262e2-5b41-43c9-95f2-9c3c447e3bd2.png)

Verify terraform state successfully move to s3

![image](https://user-images.githubusercontent.com/49937302/126871002-fb7a76f3-d557-4852-b39f-ce53d4b83ca1.png)

![image](https://user-images.githubusercontent.com/49937302/126871030-679827c6-7ca0-4615-88a0-25b3dcccaf5b.png)

Navigate to the DynamoDB table inside AWS and leave the page open in your browser. Run terraform plan and while that is running, refresh the browser and see how the lock is being handled:

![image](https://user-images.githubusercontent.com/49937302/126871109-23b0db1f-e7c4-4787-b438-f73e8cabcaa0.png)

After terraform plan completes, refresh DynamoDB table.

![image](https://user-images.githubusercontent.com/49937302/126871030-679827c6-7ca0-4615-88a0-25b3dcccaf5b.png)

Add Terraform Output

Before you run terraform apply let us add an output so that the S3 bucket Amazon Resource Names ARN and DynamoDB table name can be displayed.

Create a new file and name it output.tf and add below code.

    output "s3_bucket_arn" {
      value       = aws_s3_bucket.terraform_state.arn
      description = "The ARN of the S3 bucket"
    }
    output "dynamodb_table_name" {
      value       = aws_dynamodb_table.terraform_locks.name
      description = "The name of the DynamoDB table"
    }

run terraform apply

Terraform will automatically read the latest state from the S3 bucket to determine the current state of the infrastructure. Even if another engineer has applied changes, the state file will always be up to date.

Now, head over to the S3 console again, refresh the page, and click the grey “Show” button next to “Versions.” You should now see several versions of your terraform.tfstate file in the S3 bucket:

![image](https://user-images.githubusercontent.com/49937302/126871353-8b264a34-5842-470f-9fbb-08f6b35d3877.png)

# Security Groups refactoring with dynamic block

    resource "aws_security_group" "bastion_sg" {
      name        = "vpc_web_sg"
      description = "Allow incoming ssh & HTTP connections."
      vpc_id      = aws_vpc.main.id

      dynamic "ingress" {
        for_each = var.bastion_ingress_port
        iterator = port # set a name variables else defaults to ingress
        content {
          from_port   = port.value
          to_port     = port.value
          protocol    = var.tcp_protocol
          cidr_blocks = var.allow_ipv4_anywhere
        }

      }

      egress {
        from_port   = 0
        to_port     = 0
        protocol    = var.tcp_protocol
        cidr_blocks = var.allow_ipv4_anywhere
      }

      tags = {
        Name        = "Bastion-SG"
        Environment = var.environment
      }
    }

    resource "aws_security_group" "my-alb-sg" {
      name   = "my-alb-sg"
      description = "SG-MyALB"
      vpc_id = aws_vpc.main.id

      dynamic "ingress" {
        for_each = var.alb_ingress_port
        iterator = port
        content {
          from_port = port.value
          to_port = port.value
          protocol = var.tcp_protocol
          cidr_blocks = var.allow_ipv4_anywhere
        }
      }

      egress {
        from_port   = 0
        to_port     = 0
        protocol    = var.allow_all_traffic
        cidr_blocks = var.allow_ipv4_anywhere
      }
    }

    resource "aws_security_group" "my-asg-sg" {
      name   = "my-asg-sg"
      vpc_id = aws_vpc.main.id

      dynamic "ingress" {
        for_each = var.asg_ingress_port
        iterator = port
        content {
          from_port = port.value
          to_port = port.value
          protocol = var.tcp_protocol
          cidr_blocks = var.allow_ipv4_anywhere
        }
      }

      egress {
        from_port = 0
        to_port = 0
        protocol = var.allow_all_traffic
        cidr_blocks = var.allow_ipv4_anywhere
      }
    }

    resource "aws_security_group" "private_sg" {
        name = "vpc_private"
        description = "Allow incoming HTTP connections."
        vpc_id = aws_vpc.main.id

        dynamic "ingress" {
          for_each = var.private_ingress_port
          iterator = port
          content {
            from_port   = port.value
            to_port     = port.value
            protocol    = var.tcp_protocol
            cidr_blocks = var.allow_ipv4_anywhere
          }
        }

        egress {
          from_port   = 0
          to_port     = 0
          protocol    = var.allow_all_traffic
          cidr_blocks = var.allow_ipv4_anywhere
        }

        tags = {
            Name = "private-sg"
        }


    }

    resource "aws_security_group" "SG" {
      vpc_id      = aws_vpc.main.id
      name        = "SG"
      description = "Allow Inbound Traffic"

      dynamic "ingress" {
        for_each = var.sg_ingress_port
        iterator = port
        content{
          from_port   = port.value
          to_port     = port.value
          protocol    = var.tcp_protocol
          cidr_blocks = [var.vpc_cidr]
        }
      }

        ingress {
        from_port   = 80
        to_port     = 80
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
      }

      egress {
        from_port = 0
        to_port   = 0
        protocol  = "-1"
        cidr_blocks = [var.vpc_cidr]
      }

      dynamic "egress" {
        for_each = var.sg_egress_port
        iterator = port
        content {
          from_port = port.value
          to_port = port.value
          protocol = var.tcp_protocol
          cidr_blocks = var.allow_ipv4_anywhere
        }
      }

      tags =  {
        Name = "efs-SG"
      }

    }

    resource "aws_security_group" "myapp_mysql_rds" {
      name        = "secuirty_group_web_mysqlserver"
      description = "Allow access to MySQL RDS"
      vpc_id      = aws_vpc.main.id

      dynamic "ingress" {
        for_each = var.rds_ingress_port
        iterator = port
        content {
          from_port = port.value
          to_port = port.value
          protocol = var.tcp_protocol
          cidr_blocks = [var.vpc_cidr]
        }
      }

      tags = {
        Name = "rds_security_group"
      }

    }



# EC2 refactoring with Map and Lookup

mapping ami to respective region to make it dynamic

    variable "images" {
        type = "map"
        default = {
            ap-southeast-1 = "ami-0d058fe428540cd89"
            ap-southeast-2 = "ami-0567f647e75c7bc05"
        }
    }

use lookup function to map to the region

    resource "aws_instance" "bastion" {
      count                       = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets
      key_name                    = aws_key_pair.vincent.key_name
      ami                         = lookup(var.images,var.region,var.ami)

# Terraform Modules and best practices to structure your .tf codes

Modules serve as containers that allow to logically group Terraform codes for similar resources in the same domain (e.g., Compute, Networking, AMI, etc.). One root module can call other child modules and insert their configurations when applying Terraform config. This concept makes your code structure neater, and it allows different team members to work on different parts of configuration at the same time.

You can also create and publish your modules to Terraform Registry for others to use and use someone’s modules in your projects.

Module is just a collection of .tf and/or .tf.json files in a directory.

# Refactor project using Modules

