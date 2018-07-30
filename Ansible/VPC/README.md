# Prerequisities

- IAM user
- Boto Installed

## 1. Create a custom Role

Firstly, let's run `ansible-galaxy init aws-vpc`

We would be using two files:

```
- tasks:
  - main.yml
- vars:
  - main.yml
```

### Variables

Let's place them in `vars/main.yml`:

```
---
aws_access_key: " "
aws_secret_key: " "
region: "us-east-2"

#VPC
vpc_cidr: <PICK> (F.E. 10.10.0.0/24)
vpc_name: "Ansible VPC"

#Subnet
subnet_name: "Ansible Subnet"
subnet_cidr: <PICK> (F.E. 10.10.0.0/26)

#Internet Gateway Name
igw_name: "Traffic IGW"

securitygroup_name: "Ansible Security Group"

ec2_tag: "WebServer"

ec2_key_directory: "/home/ansible/roles/aws-vpc/"
keypair_name: "keypair"
```

## 2. Create the VPC

We will include:
* VPC
* Subnet
* Router
* IGW (Internet Gateway)
* Security Group
* EC2 Instance

### a) VPC

```
- name: create VPC
  ec2_vpc_net:
    name: "{{ VPC_NAME }}"
    cidr_block: "{{ VPC_CIDR }}"
    region: "{{ region }}"
    state: present
    state: present
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
  register: vpc
```

### b) Subnet

```
- name: associate subnet to the VPC
  ec2_vpc_subnet:
    state: present
    vpc_id: "{{ vpd_id }}"
    region: "{{ region }}"
    cidr: "{{ subnet_cidr }}"
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    map_public: yes
    resource_tags:
      name: "{{ subnet_name }}"
    register: subnet

- name: create IGW
  ec2_vpc_igw:
    vpc_id: "{{ vpc_id }}"
    region: "{{ region }}"
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    state: "present"
    tags:
      name: "{{ igw_name }}"
    register: igw
```

### c) Router

```
- name: Route IGW
  ec2_vpc_route_table:
    vpc_id: "{{ vpc_id }}"
    region: "{{ region }}"
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    subnets:
      - "{{ subnet.subnet.id }}"
    routes:
      - dest: 0.0.0.0/0
        gateway_id: "{{ igw.gateway_id }}"
    tags:
      name: "{{ route_name }}"

#update the CIDR address in the SSH port section

- name: Create Security Group
  ec2_group:
    name: <NAME>
    description: <DESCRIPTION>
    vpc_id: "{{ vpc_id }}"
    region: "{{ region }}"
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    rules:
      - proto: tcp
        ports:
        - 80
        cidr_ip: 0.0.0.0/0
      - proto: tcp
        ports:
        - 22
        cidr_ip: 0.0.0.0/0
    register: security_group
```

### d) Create EC2 key pair

```
- name: cretae a new EC2 key pair
  ec2_key:
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    name: ec2_keypair
    region: "{{ region }}"
  register: keypair

- name: Copy EC2 Private Key Locally so it can be later used
  copy: content="{{ keypair.key.private_key }}" dest={{ ec2_key_directory}} key.ppk
  when: keypair.changed == true

- name: create ec2
  ec2:
    image: ami-467ca739
    wait: yes
    instance_type: t2.micro
    region: "{{ region }}"
    group_id: "{{ security_group.group_id }}"
    vpc_subnet_id: "{{ subnet.subnet.id }}"
    key_name: "{{ keypair.key.name }}"
    count_tag:
      name: apacheserver
    exact_count: 1
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
```
