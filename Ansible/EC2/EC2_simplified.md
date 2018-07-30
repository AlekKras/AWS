# Intro

We will be using CentOS 7 as the Ansible Host.
First of all, let's run the following commands:

```
sudo pip install --upgrade pip
sudo pip install boto
sudo yum install ansible
```

After that, log in to your account. You should have
`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.

Let's say that those are our credentials:

| Name | Value |
| :--- | :--- |
| Access Key ID | NUHKOIJFOJF9GFJDO |
| Secret Access Key | LSDJKFODSJF9SDJF8UH3U3HFKW |

<sub><sup> credentials are randomly generated </sup></sub>

Let's say we want to store them locally (to be safe):

```
export AWS_ACCESS_KEY_ID="NUHKOIJFOJF9GFJDO"
export AWS_SECRET_ACCESS_KEY="LSDJKFODSJF9SDJF8UH3U3HFKW"
```
Nice, we have those. Sweet!

# Host file

Create the `~/hosts` file with the following contents:

```
[local]
localhost

[webserver]
```

Sweet, that was easy...

# EC2

Now we build Ansible file. Put the following file `~/ec2-basic.yml`

```
---
  - name: Provision EC2
    hosts: local
    connection: local
    gather_facts: False
    tags: provisioning
    vars:
      instance_type: t2.micro
      security_group: ansible_webserver #Change the name here
      image: ami-719fb712 #that's whatever image was available, change it
      keypair: agix_key #that's the name of the key, change it
      region: us-east-1 #change for sure
      count: 1

    tasks:

      - name: Create a security group
        local_action:
          module: ec2_group
          name: "{{ security_group }}"
          description: <PUT WHATEVER>
          region: "{{ region }}"
          rules:
            - proto: tcp
              from_port: 22
              to_port: 22
              cidr_ip: 0.0.0.0/0
            - proto: tcp
              from_port: 80
              to_port: 80
              cidr_ip: 0.0.0.0/0
            - proto: tcp
              from_port: 443
              to_port: 443
              cidr_ip: 0.0.0.0/0
          rules_egress:
            - proto: all
              cidr_ip: 0.0.0.0/0
        register: basic_firewall


      - name: Launch the new EC2 instance
        local_action: ec2
                    group={{ security_group }}
                    instance_type={{ instance_type}}
                    image={{ image }}
                    wait=true
                    region={{ region }}
                    keypair={{ keypair }}
                    count={{count}}

        register: ec2

      - name: add ec2 to the local host group
        local_action: lineinfile
                      dest="./hosts"
                      regexp={{ item.public_ip }}
                      insertafter="[webserver]" line={{ item.public_ip }}
        with_items: "{{ ec2.instances }}"

      - name: wait for SSH to come up
        local_action: wait_for
                      host={{ item.public_ip }}
                      port=22
                      state=started
        with_items: "{{ ec2.instances }}""

      - name: Add tag to Instance(s)
        local_action: ec2_tag resource={{ item.id }} region={{ region }} state=present
        with_items: "{{ ec2.instances }}"
        args:
          tags:
            Name: webserver
```

Now we need to bring the provisioning:

`ansible-playbook -i ./hosts ec2-basic.yml`

and finally log into your ec2 instance:

`ssh -1 centos 53.1.2.3 -i Downloads/agix.key.pem`

