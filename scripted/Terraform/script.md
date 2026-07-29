Terraform script to create an AWS EC2 instance, useful for both practice and interviews.
```
Folder structure


ec2-terraform/
├── provider.tf
├── main.tf
├── variables.tf
└── outputs.tf

provider.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

variables.tf

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-south-1"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

main.tf

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }

  filter {
    name   = "architecture"
    values = ["x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_instance" "web_server" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type

  tags = {
    Name        = "terraform-ec2"
    Environment = "dev"
  }
}

outputs.tf

output "instance_id" {
  value = aws_instance.web_server.id
}

output "public_ip" {
  value = aws_instance.web_server.public_ip
}

Then run:

aws configure

terraform init
terraform fmt
terraform validate
terraform plan
terraform apply


```
To remove the instance:

terraform destroy

Interview explanation

First → I configure the AWS provider and region.

Next → I use an AMI data source so I don't hard-code an AMI ID that varies by region.

Then → I create the EC2 instance using aws_instance, passing the AMI and instance type.

Finally → I expose useful values such as the instance ID and public IP through Terraform outputs.

For a production-style EC2 setup, I'd normally add VPC/subnet, security groups, IAM role, EBS configuration, SSH/SSM access, remote S3 state, DynamoDB state locking, and reusable Terraform modules.
