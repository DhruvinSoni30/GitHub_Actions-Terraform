# Create various resources on AWS using Terraform & GitHub Actions

### What is GitHub Actions?

GitHub Actions help you automate tasks within your software development life cycle. GitHub Actions are event-driven, meaning that you can run a series of commands after a specified event has occurred. For example, every time someone creates a pull request for a repository or someone merges the code to the repository, you can automatically run a command that executes a software testing script.

![1](https://github.com/DhruvinSoni30/GitHub_Actions-Terraform/blob/main/1.jpeg)

### What is Terraform?

Terraform is a free and open-source infrastructure as code (IAC) that can help to automate the deployment, configuration, and management of the remote servers. Terraform can manage both existing service providers and custom in-house solutions.

![2](https://github.com/DhruvinSoni30/GitHub_Actions-Terraform/blob/main/2.png)

In this tutorial, I have integrated Terraform with GitHub Actions and created various resources on AWS.

![3](https://github.com/DhruvinSoni30/GitHub_Actions-Terraform/blob/main/3.jpeg)

### Prerequisite:

* The GitHub Account
* Basic understanding of Terraform, GitHub Actions & AWS
* An Access key & Secret key created the AWS

Lets, start with the configuration of the project

**Step 1:- Create a Repository**

* Create a repository in your GitHub account & give it a name of your choice
* Create below folder inside the root folder

`.github/workflows` → In this directory, we will have a GitHub action .yml file.

![4](https://github.com/DhruvinSoni30/GitHub_Actions-Terraform/blob/main/4.png)

**Step 2:- Create a terraform file for resource creation**

* Create a `.tf` file inside the root directory and add the below code in it.
  ```
  # Configure and downloading plugins for aws
  provider "aws" {
    region     = "${var.aws_region}"
  } 

  # Creating VPC
  resource "aws_vpc" "demovpc" {
    cidr_block       = "${var.vpc_cidr}"
    instance_tenancy = "default"

    tags = {
      Name = "Demo VPC"
    }
  }

  # Creating Internet Gateway 
  resource "aws_internet_gateway" "demogateway" {
    vpc_id = "${aws_vpc.demovpc.id}"
  }

  # Grant the internet access to VPC by updating its main route table
  resource "aws_route" "internet_access" {
    route_table_id         = "${aws_vpc.demovpc.main_route_table_id}"
    destination_cidr_block = "0.0.0.0/0"
    gateway_id             = "${aws_internet_gateway.demogateway.id}"
  }

  # Creating 1st subnet 
  resource "aws_subnet" "demosubnet" {
    vpc_id                  = "${aws_vpc.demovpc.id}"
    cidr_block             = "${var.subnet_cidr}"
    map_public_ip_on_launch = true
    availability_zone = "us-east-1a"

    tags = {
      Name = "Demo subnet"
    }
  }

  # Creating 2nd subnet 
  resource "aws_subnet" "demosubnet1" {
    vpc_id                  = "${aws_vpc.demovpc.id}"
    cidr_block             = "${var.subnet1_cidr}"
    map_public_ip_on_launch = true
    availability_zone = "us-east-1b"

    tags = {
      Name = "Demo subnet 1"
    }
  }

  # Creating Security Group
  resource "aws_security_group" "demosg" {
    name        = "Demo Security Group"
    description = "Demo Module"
    vpc_id      = "${aws_vpc.demovpc.id}"

  # Inbound Rules
  # HTTP access from anywhere
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTPS access from anywhere
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # SSH access from anywhere
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Splunk default port
  ingress {
    from_port   = 8000
    to_port     = 8000
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  # Replication Port
  ingress {
    from_port   = 8089
    to_port     = 8089
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Management Port
  ingress {
    from_port   = 4598
    to_port     = 4598
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Ingestion Port
  ingress {
    from_port   = 9997
    to_port     = 9997
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Outbound Rules
  # Internet access to anywhere
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  }

  # Creating key pair
  resource "aws_key_pair" "demokey" {
    key_name   = "${var.key_name}"
    public_key = "${file(var.public_key)}"
  }
  
  # Creating EC2 Instance
  resource "aws_instance" "demoinstance" {

  # AMI based on region 
  ami = "${lookup(var.ami, var.aws_region)}"

  # Launching instance into subnet 
  subnet_id = "${aws_subnet.demosubnet.id}"

  # Instance type 
  instance_type = "${var.instancetype}"
  
  # Count of instance
  count= "${var.master_count}"
  
  # SSH key that we have generated above for connection
  key_name = "${aws_key_pair.demokey.id}"

  # Attaching security group to our instance
  vpc_security_group_ids = ["${aws_security_group.demosg.id}"]

  # Attaching Tag to Instance 
  tags = {
    Name = "Search-Head-${count.index + 1}"
  }
  
  # Root Block Storage
  root_block_device {
    volume_size = "40"
    volume_type = "standard"
  }
  
  #EBS Block Storage
  ebs_block_device {
    device_name = "/dev/sdb"
    volume_size = "80"
    volume_type = "standard"
    delete_on_termination = false
  }
  
  # SSH into instance 
  connection {
    
    # Host name
    host = self.public_ip
    # The default username for our AMI
    user = "ec2-user"
    # Private key for connection
    private_key = "${file(var.private_key)}"
    # Type of connection
    type = "ssh"
  }
  
  # Installing splunk on newly created instance
  provisioner "remote-exec" {
    inline = [
      "sudo yum update -y",
      "sudo amazon-linux-extras install docker -y",
      "sudo service docker start",
      "sudo usermod -a -G docker ec2-user",
      "sudo chkconfig docker on",
      "sudo yum install -y git",
      "sudo chmod 666 /var/run/docker.sock",
      "docker pull dhruvin30/dhsoniweb:v1",
      "docker run -d -p 80:80 dhruvin30/dhsoniweb:latest"   
  ]
  }
  }
  ```
  
**Step 3:- Create terraform file for variables**

* In the resource creation file, I have used multiple variables so that we need to create a variable file that contains the definition of the variables
* Create a `vars.tf` file in the root directory and add the below code to it
* The below code will create various resources on AWS.

  ```
  # Defining Public Key
  variable "public_key" {
    default = "tests.pub"
  }

  # Defining Private Key
  variable "private_key" {
    default = "tests.pem"
  }

  # Definign Key Name for connection
  variable "key_name" {
    default = "tests"
    description = "Desired name of AWS key pair"
  }

  # Defining Region
  variable "aws_region" {
    default = "us-east-1"
  }

  # Defining CIDR Block for VPC
  variable "vpc_cidr" {
    default = "10.0.0.0/16"
  }

  # Defining CIDR Block for Subnet
  variable "subnet_cidr" {
    default = "10.0.1.0/24"
  }

  # Defining CIDR Block for 2d Subnet
  variable "subnet1_cidr" {
    default = "10.0.2.0/24"
  }

  # Defining AMI
  variable "ami" {
    default = {
      eu-west-1 = "ami-0ea3405d2d2522162"
      us-east-1 = "ami-09d95fab7fff3776c"
    }
  }

  # Defining Instace Type
  variable "instancetype" {
    default = "t2.medium"
  }

  # Defining Master count 
  variable "master_count" {
    default = 1
  } 
  ```
**Step 4:- Store the keys**

* In order to create the resources in the AWS account, we must need to have the AWS Access Key & AWS Secret Key
* Now, we need to store the AWS Access Key & AWS Secret Key in the secrets section of the repository

![5](https://github.com/DhruvinSoni30/GitHub_Actions-Terraform/blob/main/5.png)

**Step 5:- Create a workflow file**

* Now in order to create the terraform resources automatically, we need to create a workflow file inside the `.github/workflow` directory
* Create a `.yml` file and add the below code to it
* The below job will run on every push and pull request that happens on the main branch. In the build section, I have specified the image name and commands in the run section.

  ```
  name: Terraform-GitHub-Actions
  on:
    push:
      branches: [ main ]
    pull_request:
      branches: [ main ]
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.aws_access_key }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.aws_secret_key }}
  jobs:
    build:
      name: build
      runs-on: ubuntu-latest
      steps:
        - name: Checkout
          uses: actions/checkout@v2
       
        - name: Set up Terraform
          uses: hashicorp/setup-terraform@v1
      
        - name: Terraform Init
          id: init
          run: terraform init
      
        - name: Terraform Plan
          id: plan
          run: terraform plan
      
        - name: Terraform Apply
          id: apply
          run: terraform apply --auto-approve
  ```      

**Step 6:- Check the output**

* Now, As soon as you commit your workflow file GitHub will trigger the action and the resources will be going to create on the AWS account.

![6](https://github.com/DhruvinSoni30/GitHub_Actions-Terraform/blob/main/6.png)

* After running the job you will see that all the steps run perfectly and there was no error. So you will have a grey color tick mark as each step run successfully.

![7](https://github.com/DhruvinSoni30/GitHub_Actions-Terraform/blob/main/7.png)

* You can also check the output of each step by expanding it

![8](https://github.com/DhruvinSoni30/GitHub_Actions-Terraform/blob/main/8.png)

**Step 7:- Check the resources on AWS**

1. VPC
2. Subnets
3. Internet Gateway
4. Route Table
5. Security Group
6.  Key Pair
7.  EC2 Instance

**Step 8:- Verify the output**

* In the Terraform configuration code, we have used Remote Provisioner to install docker and also ran the docker image on it
* In order to check the output navigate to `<public-ip:80>`
* You should see an output like below

![9](https://github.com/DhruvinSoni30/GitHub_Actions-Terraform/blob/main/9.png)

That’s it now, you have learned how to integrate Terraform with GitHub actions. You can now play with it and modify it accordingly.
