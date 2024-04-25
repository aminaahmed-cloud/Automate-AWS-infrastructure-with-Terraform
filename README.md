# Automate AWS Infrastructure with Terraform

This project provides a step-by-step guide to automate AWS infrastructure provisioning using Terraform. Below are the detailed steps involved:

## 1. VPC & Subnets

** Create main.tf**

```hcl
provider "aws" {
  region = "eu-west-2"
}

variable "vpc_cidr_block" {}
variable "subnet_cidr_block" {}
variable "avail_zone" {}
variable "env_prefix" {}

resource "aws_vpc" "myapp_vpc" {
  cidr_block = var.vpc_cidr_block
  tags = {
    Name = "${var.env_prefix}-vpc"
  }
}

resource "aws_subnet" "myapp_subnet_1" {
  vpc_id            = aws_vpc.myapp_vpc.id
  cidr_block        = var.subnet_cidr_block
  availability_zone = var.avail_zone
  tags = {
    Name = "${var.env_prefix}-subnet-1"
  }
}
```

**Create terraform.tfvars**

```
vpc_cidr_block = "10.0.0.0/16"
subnet_cidr_block = "10.0.10.0/24"
avail_zone = "eu-west-2a"
env_prefix = "dev"
```

<img src="https://i.imgur.com/XUC21XT.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

---

<img src="https://i.imgur.com/VnG5nLc.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


## 2. Route Table & Internet Gateway

- Create a new route table and associate it with the VPC.

```hcl
resource "aws_route_table" "myapp-route-table" {
  vpc_id = aws_vpc.myapp_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.myapp-igw.id
  }

  tags = {
    Name = "${var.env_prefix}-rtb"
  }
}

resource "aws_internet_gateway" "myapp-igw" {
  vpc_id = aws_vpc.myapp_vpc.id
  tags = {
    Name = "${var.env_prefix}-igw"
  }
}
```
- Execute the following command to apply the changes:

```
%terraform apply -auto-approve
```

## 3. Subnet Association with Route Table

- Associate the subnet with the route table created.

```hcl
resource "aws_route_table_association" "a-rtb-subnet" {
  subnet_id       = aws_subnet.myapp_subnet_1.id
  route_table_id  = aws_route_table.myapp-route-table.id
}
```

<img src="https://i.imgur.com/78F1ciY.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>



## 4. Use the Main Route Table

- Get the default route table ID and use it for the main route table.

```hcl
resource "aws_default_route_table" "main-rtb" {
  default_route_table_id = aws_vpc.myapp_vpc.default_route_table_id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.myapp-igw.id
  }

  tags = {
    Name = "${var.env_prefix}-main-rtb"
  }
}
```

- Execute the following command to apply the changes:

```bash
terraform apply -auto-approve
```

<img src="https://i.imgur.com/n5PyaMU.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


## 5. Security Group (Firewall)

- Create a security group to control inbound and outbound traffic.

```hcl

resource "aws_security_group" "myapp-sg" {
  name        = "myapp-sg"
  vpc_id      = aws_vpc.myapp_vpc.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "TCP"
    cidr_blocks = [var.my_ip]
  }

  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "TCP"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port       = 0
    to_port         = 0
    protocol        = "-1"
    cidr_blocks     = ["0.0.0.0/0"]
    prefix_list_ids = []
  }
  
  tags = {
    Name = "${var.env_prefix}-sg"
  }
}
```
- Execute the following command to apply the changes:

```bash

%terraform apply -auto-approve
```

<img src="https://i.imgur.com/nLR3IKY.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


## 6. Use Default Security Group

- Use the default security group if needed.

<img src="https://i.imgur.com/mqsjPlt.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

---

<img src="https://i.imgur.com/DrBoVsp.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


## 7. Amazon Machine Image for EC2

- Add the following code to your main.tf file:

```hcl
data "aws_ami" "latest-amazon-linux-image" {
  most_recent = true
  owners = ["amazon"]
  filter {
    name = "name"
    values = ["amzn2-ami-kernel-*-x86_64-gp2"]
  }
  filter {
    name = "virtualization-type"
    values = ["hvm"]
  }
}

output "aws_ami_id" {
  value = data.aws_ami.latest-amazon-linux-image.id
}
```

Then apply the changes:

```
terraform apply -auto-approve
```

<img src="https://i.imgur.com/PmernQ7.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

---

<img src="https://i.imgur.com/8LCorA0.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


## 8. Create EC2 Instance

Generate the key pair server-key-pair.pem and move it to ~/.ssh/. Change its permissions with:

```bash
chmod 400 ~/.ssh/server-key-pair.pem
```

Edit your main.tf file with the following content:

```hcl
resource "aws_instance" "myapp-server" {
  ami                         = data.aws_ami.latest-amazon-linux-image.id
  instance_type               = var.instance_type
  subnet_id                   = aws_subnet.myapp_subnet_1.id
  vpc_security_group_ids      = [aws_default_security_group.default-sg.id]
  availability_zone           = var.avail_zone
  associate_public_ip_address = true
  key_name                    = "server-key-pair"

  tags = {
    Name = "${var.env_prefix}-server"
  }
}
```

Apply the configuration using:

```bash

terraform apply -auto-approve
```

**SSH into the EC2 instance with:**

```bash

ssh -i ~/.ssh/server-key-pair.pem ec2-user@<public_ip_address>
```

**- Automate SSH Key Pair**

To automate SSH key pair generation, add the following code to your main.tf file:

```hcl

resource "aws_key_pair" "ssh-key" {
  key_name   = "server-key"
  public_key = file(var.public_key_location)
}

resource "aws_instance" "myapp-server" {
  ami                         = data.aws_ami.latest-amazon-linux-image.id
  instance_type               = var.instance_type
  subnet_id                   = aws_subnet.myapp_subnet_1.id
  vpc_security_group_ids      = [aws_default_security_group.default-sg.id]
  availability_zone           = var.avail_zone
  associate_public_ip_address = true
  key_name                    = aws_key_pair.ssh-key.key_name

  tags = {
    Name = "${var.env_prefix}-server"
  }
}
```

- Apply the changes:

```bash

terraform apply -auto-approve
```

- Get the public IP address of the instance:

```bash

terraform state show aws_instance.myapp-server.public_ip
```

- SSH into the instance:

```bash

%ssh -i ~/.ssh/id_ed25519 ec2-user@<public_ip_address>
```

**Then, remove the temporary key pair:**

```bash
rm ~/.ssh/server-key-pair.pem
```


<img src="https://i.imgur.com/8rCPzz7.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

---

<img src="https://i.imgur.com/sSXBYcQ.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

