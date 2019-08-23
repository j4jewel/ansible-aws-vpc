# ansible-aws-vpc
Playbook to create a VPC in AWS with public and private subnets


```sh
---
- name: Creating VPC
  hosts: localhost

  vars:
    aws_region: us-east-1
    vpc_name: Ansible_vpc
    vpc_cidr_block: 172.16.0.0/16
    public_subnet_1_cidr: 172.16.0.0/20
    public_subnet_2_cidr:  172.16.16.0/20
    private_subnet_1_cidr: 172.16.32.0/20
  tasks:

    - name: Creating VPC
      ec2_vpc_net:
        name:       "{{ vpc_name }}"
        cidr_block: "{{ vpc_cidr_block }}"
        region:     "{{ aws_region }}"
        state:      "present"
      register: my_vpc

    - name: Wait for 15 Seconds
      wait_for: timeout=15

    - name: Set VPC ID in variable
      set_fact:
        vpc_id: "{{ my_vpc.vpc.id }}"

    - name: Creating Public Subnet [AZ-1]
      ec2_vpc_subnet:
        state:  "present"
        vpc_id: "{{ vpc_id }}"
        cidr:   "{{ public_subnet_1_cidr }}"
        az:     "{{ aws_region }}a"
        region: "{{ aws_region }}"
        resource_tags:
          Name: "Public Subnet-1"
      register: my_public_subnet_az1

    - name: Set Public Subnet ID in variable [AZ-1]
      set_fact:
        public_subnet_az1_id: "{{ my_public_subnet_az1.subnet.id }}"

    - name: Creating Public Subnet [AZ-2]
      ec2_vpc_subnet:
        state:  "present"
        vpc_id: "{{ vpc_id }}"
        cidr:   "{{ public_subnet_2_cidr }}"
        az:     "{{ aws_region }}b"
        region: "{{ aws_region }}"
        resource_tags:
          Name: "Public Subnet-2"
      register: my_public_subnet_az2

    - name: Set Public Subnet ID in variable [AZ-2]
      set_fact:
        public_subnet_az2_id: "{{ my_public_subnet_az2.subnet.id }}"

    - name: Creating Private Subnet [AZ-1]
      ec2_vpc_subnet:
        state:  "present"
        vpc_id: "{{ vpc_id }}"
        cidr:   "{{ private_subnet_1_cidr }}"
        az:     "{{ aws_region }}c"
        region: "{{ aws_region }}"
        resource_tags:
          Name: "Private Subnet-1"
      register: my_private_subnet_az3

    - name: Set Private Subnet ID in variable [AZ-3]
      set_fact:
        private_subnet_az3_id: "{{ my_private_subnet_az3.subnet.id }}"

    - name: Create Internet Gateway for VPC
      ec2_vpc_igw:
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        state:  "present"
      register: my_vpc_igw

    - name: Set Internet Gateway ID in variable
      set_fact:
        igw_id: "{{ my_vpc_igw.gateway_id }}"

    - name: Creating a new NAT gateway and allocate new EIP if a NAT gateway does not yet exist in the subnet.
      ec2_vpc_nat_gateway:
        state: present
        subnet_id: "{{ public_subnet_az2_id }}"
        wait: yes
        region:    "{{ aws_region }}"
        if_exist_do_not_create: true
      register: new_nat_gateway_az2

    - name: Set Nat Gateway ID in variable [AZ-2]
      set_fact:
        nat_gateway_az2_id: "{{ new_nat_gateway_az2.nat_gateway_id }}"

    - name: Wait for 15 Seconds
      wait_for: timeout=15

    - name: Set up public subnet route table
      ec2_vpc_route_table:
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        tags:
          Name: "Public"
        subnets:
          - "{{ public_subnet_az1_id }}"
          - "{{ public_subnet_az2_id }}"
        routes:
          - dest: "0.0.0.0/0"
            gateway_id: "{{ igw_id }}"

    - name: Set up private subnet route table [AZ-3]
      ec2_vpc_route_table:
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        tags:
          Name: "Private"
        subnets:
          - "{{ private_subnet_az3_id }}"
        routes:
          - dest: "0.0.0.0/0"
            gateway_id: "{{ nat_gateway_az2_id }}"

    - name: Creating Main Security Group
      ec2_group:
        name:  "External SSH Access"
        description: "Bastion"
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        rules:
          - proto: tcp
            from_port: 22
            to_port: 22
            cidr_ip: 0.0.0.0/0
      register: my_main_sg

    - name: Set Main Security Group ID in variable
      set_fact:
        main_sg_id: "{{ my_main_sg.group_id }}"

    - name: Creating Private Security Group
      ec2_group:
        name: "Private Instances SG"
        description: "Private Instances SG"
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        rules:
          - proto: "tcp"
            from_port: "22"
            to_port: "22"
            group_id: "{{ main_sg_id }}"
```
