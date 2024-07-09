
# terraform-own-assignment

Remember for backend configuration. First create backend requirement like S3 bucket and dynamodb for statelocking through terraform. Once it done that add backend section in terraform for maintaining the statefile..

    main. tf
    ----
    terraform {
      required_providers {
        aws = {
          source = "hashicorp/aws"
          version = "5.57.0"
        }
      }
    
      backend "s3" {
        bucket         = "terraform-state1232"
        key            = "terraform.tfstate"
        region         = "us-west-2"
        dynamodb_table = "terraform-locks"
        encrypt        = true
      }
    }
    
    provider "aws" {
      region     = "us-west-2"
      access_key = ""
      secret_key = ""
    }
    
    # S3 Bucket for Terraform state
    resource "aws_s3_bucket" "terraform_state" {
      bucket = "terraform-state1232"
    
      versioning {
        enabled = true
      }
    
      lifecycle {
        prevent_destroy = true
      }
    
      tags = {
        Name        = "terraform-state"
        Environment = "dev"
      }
    }
    
    # DynamoDB table for state locking
    resource "aws_dynamodb_table" "terraform_locks" {
      name         = "terraform-locks"
      billing_mode = "PAY_PER_REQUEST"
      hash_key     = "LockID"
    
      attribute {
        name = "LockID"
        type = "S"
      }
    
      tags = {
        Name        = "terraform-locks"
        Environment = "dev"
      }
    }
    
    # VPC
    resource "aws_vpc" "main" {
      cidr_block = "10.0.0.0/16"
    }
    
    # Subnets
    resource "aws_subnet" "public_subnet1" {
      vpc_id                  = aws_vpc.main.id
      cidr_block              = "10.0.1.0/24"
      availability_zone       = "us-west-2a"
      map_public_ip_on_launch = true
    }
    
    resource "aws_subnet" "public_subnet2" {
      vpc_id                  = aws_vpc.main.id
      cidr_block              = "10.0.2.0/24"
      availability_zone       = "us-west-2b"
      map_public_ip_on_launch = true
    }
    
    resource "aws_subnet" "private_subnet1" {
      vpc_id            = aws_vpc.main.id
      cidr_block        = "10.0.3.0/24"
      availability_zone = "us-west-2a"
    }
    
    resource "aws_subnet" "private_subnet2" {
      vpc_id            = aws_vpc.main.id
      cidr_block        = "10.0.4.0/24"
      availability_zone = "us-west-2b"
    }
    
    # Internet Gateway
    resource "aws_internet_gateway" "main" {
      vpc_id = aws_vpc.main.id
    }
    
    # Route Tables
    resource "aws_route_table" "public" {
      vpc_id = aws_vpc.main.id
    
      route {
        cidr_block = "0.0.0.0/0"
        gateway_id = aws_internet_gateway.main.id
      }
    }
    
    resource "aws_route_table_association" "public_1" {
      subnet_id      = aws_subnet.public_subnet1.id
      route_table_id = aws_route_table.public.id
    }
    
    resource "aws_route_table_association" "public_2" {
      subnet_id      = aws_subnet.public_subnet2.id
      route_table_id = aws_route_table.public.id
    }
    
    # Security Groups
    resource "aws_security_group" "web_sg" {
      vpc_id = aws_vpc.main.id
    
      ingress {
        from_port   = 80
        to_port     = 80
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
      }
    
      ingress {
        from_port   = 443
        to_port     = 443
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
      }
    
      ingress {
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["39.45.191.6/32"]  # Replace with your IP
      }
    
      egress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
      }
    }
    
    resource "aws_security_group" "rds_sg" {
      vpc_id = aws_vpc.main.id
    
      ingress {
        from_port       = 3306
        to_port         = 3306
        protocol        = "tcp"
        security_groups = [aws_security_group.web_sg.id]
      }
    
      egress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
      }
    }
    
    # Key Pair
    resource "aws_key_pair" "deployer_key" {
      key_name   = "deployer-key"
      public_key = file("${path.module}/my-key.pub")
    }
    
    # EC2 Instance
    resource "aws_instance" "web" {
      ami                         = "ami-0604d81f2fd264c7b"  # Change to your preferred AMI
      instance_type               = "t2.micro"
      subnet_id                   = aws_subnet.public_subnet1.id
      security_groups             = [aws_security_group.web_sg.id]
      associate_public_ip_address = true
      key_name                    = aws_key_pair.deployer_key.key_name
    
      provisioner "local-exec" {
        command = "echo ${aws_instance.web.public_ip} > ip_address.txt"
      }
    
      tags = {
        Name = "web-server"
      }
    
      depends_on = [
        aws_security_group.web_sg,
        aws_subnet.public_subnet1,
        aws_internet_gateway.main,
        aws_route_table_association.public_1
      ]
    }
    
    # RDS Instance
    resource "aws_db_instance" "default" {
      allocated_storage    = 20
      engine               = "mysql"
      engine_version       = "5.7"
      instance_class       = "db.t3.micro"
      db_name              = "mydb"
      username             = "admin"
      password             = "admin12345"  # Change this to your password
      db_subnet_group_name = aws_db_subnet_group.main.name
      vpc_security_group_ids = [aws_security_group.rds_sg.id]
      skip_final_snapshot  = true
    
      tags = {
        Name = "rds-instance"
      }
    
      depends_on = [
        aws_security_group.rds_sg,
        aws_db_subnet_group.main
      ]
    }
    
    resource "aws_db_subnet_group" "main" {
      name       = "main"
      subnet_ids = [aws_subnet.private_subnet1.id, aws_subnet.private_subnet2.id]
    
      tags = {
        Name = "main"
      }
    }
    
    # Load Balancer
    resource "aws_lb" "web_lb" {
      name               = "web-lb"
      internal           = false
      load_balancer_type = "application"
      security_groups    = [aws_security_group.web_sg.id]
      subnets            = [aws_subnet.public_subnet1.id, aws_subnet.public_subnet2.id]
    
      enable_deletion_protection = false
    
      tags = {
        Name = "web-lb"
      }
    
      depends_on = [
        aws_security_group.web_sg,
        aws_subnet.public_subnet1,
        aws_subnet.public_subnet2
      ]
    }
    
    resource "aws_lb_target_group" "web_tg" {
      name     = "web-tg"
      port     = 80
      protocol = "HTTP"
      vpc_id   = aws_vpc.main.id
    
      health_check {
        interval            = 30
        path                = "/"
        timeout             = 5
        healthy_threshold   = 5
        unhealthy_threshold = 2
        matcher             = "200"
      }
    
      tags = {
        Name = "web-tg"
      }
    }
    
    resource "aws_lb_listener" "web_listener" {
      load_balancer_arn = aws_lb.web_lb.arn
      port              = 80
      protocol          = "HTTP"
    
      default_action {
        type             = "forward"
        target_group_arn = aws_lb_target_group.web_tg.arn
      }
    }
    
    resource "aws_lb_target_group_attachment" "web_tg_attachment" {
      target_group_arn = aws_lb_target_group.web_tg.arn
      target_id        = aws_instance.web.id
      port             = 80
    
      depends_on = [
        aws_lb_target_group.web_tg,
        aws_instance.web
      ]
    }
    
    # Outputs
    output "instance_ip_addr" {
      description = "The public IP address of the web instance"
      value       = aws_instance.web.public_ip
    }
    
    output "key_pair" {
      description = "The private key used to access the instance"
      value       = aws_key_pair.deployer_key.key_name
    }


## Ansible file
    
    ---
    - hosts: web
      become: yes
      tasks:
        - name: Install Nginx
          apt:
            name: nginx
            state: present
            update_cache: yes
    
        - name: Create index.html
          copy:
            content: "Hello, World!"
            dest: /var/www/html/index.html
    
        - name: Ensure Nginx is running
          service:
            name: nginx
            state: started
            enabled: yes

# ssh key creation

    ssh-keygen -t rsa -b 2048 -f my-key


# install awscli on local machine
  
      https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
 
    
# aws command

    -1 aws configure    ---> to connect aws account from localmachine
    -2  aws sts get-caller-identity  ---> set caller indentity mean aws account detailss 
