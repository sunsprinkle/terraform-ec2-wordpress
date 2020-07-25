# Launch & Configure EC2 For WordPress Using IaaC

Those days, when you needed to buy hardware (servers, network equipment, I/O devices)
just to work on a new I.T. project, are long gone. The advent of cloud brought the ease
for all of us consumers (both, small users & corporations). Moreover, most of the 
cloud providers allow you to be able to create infra on their platform with IaaC (
Infrastructure as a Code), meaning, you chart out the required infrastructure 
components, you write a piece of code for each component, and run that code. All those 
components you specify will be created in a matter of minutes, on to the platform of
your choice.

There are plenty of tools that can help you write IaaC stack, but "Terraform" by far the most popular and leading tool in this genre, and the reason is it's simple, consistent, and general for all cloud platforms.
In this tutorial, we will show the power and the ease with which you can create a single 'Ubuntu 18' ec2 instance with terraform, and then hand this instance over to ansible for configuring it for a generic LEMP Stack for 'Wordpress'. So let's get started.

Note: ( This piece of code was written with terraform 0.11)

```
# terraform --version
Terraform v0.11.2
```

First of all terraform uses .tf extension for itself. It allows to declare variables for personal and stack related tasks, and a main module where you specify the components of the stacks. We have written few terraform files and a role based ansible-play book, are included in our working directory. Lets list the contents..

```
# ls -1
ansible.cfg
main.tf
roles
terraform.tfvars
variables.tf
wp-lemp.yml
```

Here, variables.tf is used to declare variables and their values are stored in terraform.tfvars.
There's also roles based playbook for wordpress deployment with ansible. Let's see what's in it.

```
# cat wp-lemp.yml 
---
- name: WordPress On Lemp
  hosts: all
  become: True
  gather_facts: False
  roles:
    - preup
    - nginx
    - php
    - mysql
    - wordpress
```

So as we can see we have roles that will install Nginx, PHP, MySQL, and last but not least Wordpress itself, (pretty much a standard wordpress playbook)

**Note:**  Make sure `host_key_checking`is set to False in `/etc/ansible/ansible.cfg` or create an `ansible.cfg` file in the directory where you run this playbook.

Before examining the main piece of this project, lets make a list of what we need to create in our AWS account.

------



	— VPC with CIDR of 10.0.0.0/16
	— Internet Gateway & attach to VPC
	— Public Subnet in VPC with CIDR of 10.0.1.0/24
	— Route Table with routeable traffic to Internet Gateway
	— Associate that Route Table with public Subnet
	— Security Group with incoming ports (80 http, 22 ssh) allowed for atleast our 	 local ip
	— t2.micro EC2 instance using Ubuntu 18 Bionic Beaver AMI, with public ip enabled, launched in same subnet with provided public key, and above security group attached to it.

Below is the chunks of Code associated with each part

**--- Provider Info ---**

```
# Cloud Provider: AWS

provider "aws" {
  version = "1.13.0"
  region  = "${var.aws_region}"
  profile = "${var.aws_profile}"
}

```

**---- VPC ----**

```
resource "aws_vpc" "wp_vpc" {
  cidr_block           = "${var.vpc_cidr}"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags {
    Name = "wp_vpc"
  }
}
```

**---- Gateway ----**

```
resource "aws_internet_gateway" "wp_internet_gateway" {
  vpc_id = "${aws_vpc.wp_vpc.id}"

  tags {
    Name = "wp_igw"
  }
}
```

**---- Subnet ----**

```
resource "aws_subnet" "wp_public1_sn" {
  vpc_id                  = "${aws_vpc.wp_vpc.id}"
  cidr_block              = "${var.cidrs["public1"]}"
  map_public_ip_on_launch = true
  availability_zone       = "${data.aws_availability_zones.available.names[0]}"

  tags {
    Name = "wp_public1"
  }
}
```

**---- Route Tables ----**

```
resource "aws_route_table" "wp_public_rt" {
  vpc_id = "${aws_vpc.wp_vpc.id}"

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = "${aws_internet_gateway.wp_internet_gateway.id}"
  }

  tags {
    Name = "wp_public_rt"
  }
}
```

**---- Route Tables Association ----**

```
resource "aws_route_table_association" "wp_public1_assoc" {
  subnet_id      = "${aws_subnet.wp_public1_sn.id}"
  route_table_id = "${aws_route_table.wp_public_rt.id}"
}
```

**--- Security Group ---**

```
resource "aws_security_group" "wp_dev_sg" {
  name        = "wp_dev_sg"
  description = "Dev Instance Access"
  vpc_id      = "${aws_vpc.wp_vpc.id}"

  #Rules

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["${var.localip}"]
  }
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["${var.localip}"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

**--- WP Server ---**

```
resource "aws_key_pair" "wp_auth" {
  key_name   = "${var.key_name}"
  public_key = "${file(var.public_key_path)}"
}


data "aws_ami" "ubuntu" {
    most_recent = true

    filter {
        name   = "name"
        values = ["ubuntu/images/hvm-ssd/ubuntu-bionic-18.04-amd64-server-*"]
    }
    
    filter {
        name   = "virtualization-type"
        values = ["hvm"]
    }
    
    owners = ["099720109477"] # Canonical

}


resource "aws_instance" "wp_dev" {
  instance_type = "${var.dev_instance_type}"
  ami           = "${data.aws_ami.ubuntu.id}"

  tags {
    Name = "wp_dev"
  }

  key_name               = "${aws_key_pair.wp_auth.id}"
  vpc_security_group_ids = ["${aws_security_group.wp_dev_sg.id}"]
  subnet_id              = "${aws_subnet.wp_public1_sn.id}"

  provisioner "local-exec" {
    command = <<EOD
cat <<EOF > aws_hosts
[dev]
${aws_instance.wp_dev.public_ip} ansible_user="${var.ansible_user}"
EOF
EOD
  }

  provisioner "local-exec" {
    command = "aws ec2 wait instance-status-ok --instance-ids ${aws_instance.wp_dev.id} --profile sunsprinkle && ansible-playbook -i aws_hosts ansible/wp-lemp.yml"
  }
}
```


To run this code, we'll first configure aws cli with a profile. Provide the Access Key ID , Secret Access Key & Region Name, then you're good to go.

```
# aws configure --profile sunsprinke
AWS Access Key ID [None]: 
AWS Secret Access Key [None]: 
Default region name [None]: 
Default output format [None]:
```

Now launch the terraform, this will go and download plugins for all the providers we stated in our stack

```
# terraform init

Initializing provider plugins...
- Checking for available provider plugins on https://releases.hashicorp.com...
- Downloading plugin for provider "aws" (1.13.0)...

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

Now let's check if our plan is workable

```
# terraform plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

data.aws_availability_zones.available: Refreshing state...
data.aws_ami.ubuntu: Refreshing state...

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  + aws_instance.wp_dev
      id:                                          <computed>
      ami:                                         "ami-0ac80df6eff0e70b5"
      associate_public_ip_address:                 <computed>
      availability_zone:                           <computed>
      ebs_block_device.#:                          <computed>
      ephemeral_block_device.#:                    <computed>
      get_password_data:                           "false"
      instance_state:                              <computed>
      instance_type:                               "t2.micro"
      ipv6_address_count:                          <computed>
      ipv6_addresses.#:                            <computed>
      key_name:                                    "${aws_key_pair.wp_auth.id}"
      network_interface.#:                         <computed>
      network_interface_id:                        <computed>
      password_data:                               <computed>
      placement_group:                             <computed>
      primary_network_interface_id:                <computed>
      private_dns:                                 <computed>
      private_ip:                                  <computed>
      public_dns:                                  <computed>
      public_ip:                                   <computed>
      root_block_device.#:                         <computed>
      security_groups.#:                           <computed>
      source_dest_check:                           "true"
      subnet_id:                                   "${aws_subnet.wp_public1_sn.id}"
      tags.%:                                      "1"
      tags.Name:                                   "wp_dev"
      tenancy:                                     <computed>
      volume_tags.%:                               <computed>
      vpc_security_group_ids.#:                    <computed>

...

  + aws_vpc.wp_vpc
      id:                                          <computed>
      assign_generated_ipv6_cidr_block:            "false"
      cidr_block:                                  "10.0.0.0/16"
      default_network_acl_id:                      <computed>
      default_route_table_id:                      <computed>
      default_security_group_id:                   <computed>
      dhcp_options_id:                             <computed>
      enable_classiclink:                          <computed>
      enable_classiclink_dns_support:              <computed>
      enable_dns_hostnames:                        "true"
      enable_dns_support:                          "true"
      instance_tenancy:                            <computed>
      ipv6_association_id:                         <computed>
      ipv6_cidr_block:                             <computed>
      main_route_table_id:                         <computed>
      tags.%:                                      "1"
      tags.Name:                                   "wp_vpc"


Plan: 8 to add, 0 to change, 0 to destroy.
```

After making sure, your plan is executable, all you have to do is apply that..

```
# terraform apply
Plan: 8 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: 
```

Confirm by typing yes, and it will start creating resources, under the account configured there.
At the end, after playbook done installing wordpress, copy the Pulic ip of your instance and paste it in broswer, you'll see wordpress installation page. Follow the standard steps to finish it.

By above example, we showed how simple & powerful terraform can be for creating infrastructure we need. 