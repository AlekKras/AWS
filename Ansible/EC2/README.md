# AWS Elastic Compute Cloud

In order to begin creating AWS EC2 using Ansible, you would need:
- `AWS Access Key ID` and `Secret Access Key`.
- IAM Roles:
|- `AmazonEC2FullAccess`
|- `AmazonVPCFullAccess`
|- `AmazonRDSFullAccess`

# Security group

We need to identify the region:

```
---
- hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: create a security group
      ec2_group:
        name: <NAME OF THE GROUP>
        description: <DESCRIPTION>
        region: <PICK A REGION> (F.E. us-east-2)
        aws_access_key: "<ACCESS_KEY>"
        aws_secret_key: "<SECRET_KEY>"
        rules:
          - proto: tcp
            from_port: <PORT> (F.E. 80)
            to_port: <PORT> (F.E. 80)
            cidr_ip: <CIDR> (F.E. 0.0.0.0/0)
      register: security_group
```

# EC2

```
---
- hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: create ec2 instance
      ec2:
        aws_access_key: "<ACCESS_KEY>"
        aws_secret_key: "<SECRET_KEY>"
        image: <IMAGE> (F.E. ami-caaf84af)
        instance_type: <TYPE> (F.E. t2.micro)
        group_id: security_group.group_id
        region: <REGION> (F.E. us-east-2)
        count_tag:
          name: <PICK> (F.E. apacheserver)
        exact_count: <NUMBER> (F.E. 1)
      register: ec2
```

# Launch EC2 + SSD Volume

```
---
- hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: create EC2 with SSD Volume
      ec2:
        key_name: <KEY>
        group: <GROUP>
        instance_type: <TYPE>
        image: <IMAGE>
        wait: yes
        wait_timeout: 500
        volumes:
          - device_name: /dev/xvda
            volume_type: <TYPE> (F.E. gp2)
            volume_size: <SIZE> (F.E. 8)
        group_id: security_group.group_id
        count_tag:
          name: <PICK> (F.E. apacheserver)
        exact_count: 1
```

Here are some types of volume:

| Type | Code |
| :--- | :--- |
| General Purpose SSD | `gp2` |
| Provisioned IOPS SSD | `io1` |
| Throughput Optimized HDD | `st1` |
| Cold HDD | `sc1` |

# RDS

Let's say we need RDS

```
---
- hosts: localhost
  connection: local
  gather_facts: false
  tasks:
  - name: create RDS
    rds:
      command: create
      region: <REGION> (F.E. us-east-2)
      instance_name: <NAME>
      db_engine: <PICK> (F.E. MySQL)
      size: <SIZE> (F.E. 20)
      instance_type: <TYPE> (F.E. db.t2.micro)
      username: <NAME> (F.E. admin)
      password: <PASSWORD> (F.E. password)
      tags:
        environment: <NAME>
        application: <NAME>
```     
