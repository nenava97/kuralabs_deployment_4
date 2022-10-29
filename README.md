# kuralabs_deployment_4
Terraform deployment 4

##Create VPC with Terraform in VSC
in terraform.tfstate stored variable values for aws access key and aws secret key
in main.tf file
```
# VARIABLES
variable "aws_access_key" {}
variable "aws_secret_key" {}

# PROVIDER
provider "aws" {
  access_key = var.aws_access_key
  secret_key = var.aws_secret_key
  region = "us-east-1"
  
}

# VPC
resource "aws_vpc" "test-vpc" {
  cidr_block           = "172.19.0.0/16"
  enable_dns_hostnames = "true"
}

# ELASTIC IP 
resource "aws_eip" "nat_eip_prob" {
  vpc = true

}

# SUBNET 1
resource "aws_subnet" "subnet1" {
  cidr_block              = "172.19.0.0/18"
  vpc_id                  = aws_vpc.test-vpc.id
  map_public_ip_on_launch = "true"
  availability_zone       = data.aws_availability_zones.available.names[0]
}

# INTERNET GATEWAY
resource "aws_internet_gateway" "gw_1" {
  vpc_id = aws_vpc.test-vpc.id
}

# SEC GROUP
resource "aws_security_group" "web_ssh" {
  name        = "ssh-access"
  description = "open ssh traffic"
  vpc_id      = aws_vpc.test-vpc.id

  ingress {
    from_port = 22
    to_port = 22
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]

  }
  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    "Name" : "Web server001"
    "Terraform" : "true"
  }
  
}

# DATA
data "aws_availability_zones" "available" {
  state = "available"
}
```
![Screenshot 2022-10-22 150603](https://user-images.githubusercontent.com/108698688/198812425-2cfde520-d12f-4b5a-885a-21583bcc6ef2.jpg)
![Screenshot 2022-10-22 150625](https://user-images.githubusercontent.com/108698688/198812427-f3bcd987-e6aa-494c-9615-44c2d52fc01a.jpg)
![Screenshot 2022-10-28 183516](https://user-images.githubusercontent.com/108698688/198812433-03cab743-1cae-4a6f-b3dc-1379b8c6debc.jpg)
![Screenshot 2022-10-28 234309](https://user-images.githubusercontent.com/108698688/198812435-3882937c-a051-4e5e-8cc0-e87802fa5be6.jpg)
