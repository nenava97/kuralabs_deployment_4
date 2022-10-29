# kuralabs_deployment_4 Nicole Navarrete 10.29.22

This deployment build, tests and deploys a url-shortner Flask app with Jenkins in two ways. A EC2 with Jenkins is created and configured with AWS credentials. Jenkins runs the pipeline in the Jenkinsfile, which contains Terraform commands of init, plan, to apply a terraform folder which create another EC2 instance in a default VPC for the first way and in a VPC we create in with Terraform in VSC for the second way. A bash script is then run to deploy the application code from the GitHub reposiitory with Gunicorn from the created instance.  

## First Way: Configure Jenkins server in EC2 (created in AWS) and install Terraform to build another EC2 instance from which application will be deployed.

1. Create first EC2 in public subnet on default VPC with Jenkins server.

```diff
- Issue:
- Ensure security group has HTTP 80, Custon TCP 8080, SSH 22, Custon TCP 8000 and Custon TCP 5000. When first made EC2 didn't have all and had HTTP errors later while trying to reach application webpage.
```

- Run the following script in EC2 CLI to install packages needed for Jenkins (including java).
```
#!/bin/bash
sudo apt update
sudo apt install python3.10-venv
sudo apt -y install openjdk-11-jre
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \ /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \ https://pkg.jenkins.io/debian-stable binary/ | sudo tee \ /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get -y install jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

- Configure Jenkins by navigating to http://PublicIPv4:8080.

- Add global secret text credentials for awsaccess key and aws secret key.

2. Install Terraform in Ubuntu CLI.
```
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```

3. In Jenkins, create and build multi-branch pipeline project for url-shortner attaching GitHub repo, authorizing with GitHub personal access token.

```diff
- Issue:
- Forgot to add AWS_SECRET_KEY in Jenkins.
```
![Screenshot 2022-10-22 150603](https://user-images.githubusercontent.com/108698688/198812425-2cfde520-d12f-4b5a-885a-21583bcc6ef2.jpg)

```diff
- Issue:
- Forgot to change key name in main.tf to own key.
```
![Screenshot 2022-10-22 150625](https://user-images.githubusercontent.com/108698688/198812427-f3bcd987-e6aa-494c-9615-44c2d52fc01a.jpg)

```diff
- Issue:
- After adding the destroy stage, the Jenkins build took too long to load due to resource contention had to abort and reboot EC2. 
```
![Screenshot 2022-10-28 183516](https://user-images.githubusercontent.com/108698688/198812433-03cab743-1cae-4a6f-b3dc-1379b8c6debc.jpg)


## Second Way: Create VPC with Terraform in VSC and attach to EC2 instance that will be created in pipeline build by Terraform and Jenkins in initial EC2.

1. In VSC create new folder with main.tf and terraform.tfvars.

 - In terraform.tfvars stored variable values for aws access key and aws secret key.
 
 - In main.tf file configure VPC with public subnet, internet gateway and security group ssh port 22. 
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
2. Edit files in IntTerraform folder in GitHub repo to attach the VPC just created.

- Create terraform.tfvars file with vpc_id variable.

- In SG.tf add vpc_id variable to aws_security_group resource.
```
vpc_id = var.vpc_id
```

```diff
- Issue:
- Security group of VPC was named "ssh-access", since all resource names must be unique I had to change the security group name of this EC2 to "ec2-access" so wouldn't have error.
```

- In main.tf added data source as VPC just made for resource EC2 being created.
```
variable "vpc_id" {}

data "aws_vpc" "selected" {
  id = var.vpc_id
}
```
- Also in main.tf had to add subnet_id of subnet that was created with VPC just made.

```diff
- Issue:
- Before adding this, got error that subnet id and security group were in different networks. 
```

3. In Jenkins, re-build multi-branch pipeline. 

##Add slack notification system and additional testing

1. Add slack notification system plugin to Jenkins 
Navigate to manage Jenkins and configure system to add Slack workspace (https://kura-labs.slack.com) and add secret text credential which is Slack integration token credential ID 

2. Add test in code stored in GitHub repository that will check if a get request to the application web page returns a status code HTTP 200 OK (success) and if a post request returns a HTTP 405 Method Not Allowed (error)

## Successful build!
![Screenshot 2022-10-28 234309](https://user-images.githubusercontent.com/108698688/198812435-3882937c-a051-4e5e-8cc0-e87802fa5be6.jpg)

## Improvements
- Utilize modules to create a VPC in the intTerraform folder and then reference it for EC2 module instead of creating the VPC first on our own in VSC.
- Create one security group for both the VPC and EC2 to use instead of separate ones. 
