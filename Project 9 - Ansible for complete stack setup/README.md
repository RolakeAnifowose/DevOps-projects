# Project 9 - Ansible for Complete Stack Setup
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/f5f10ea4-2077-455a-beef-86e15fcb9284)

This is a continuation of project 8 - Ansible setup

### Step 1 - Variables Update
Add the content of the bastion_setup file to vpc_setup
```
vpc_name: "Vprofile-vpc"

#VPC Range
vpcCidr: '172.20.0.0./16'

#Subnets Range
PubSub1Cidr: 172.20.1.0/24
PubSub2Cidr: 172.20.2.0/24
PubSub3Cidr: 172.20.3.0/24
PrivSub1Cidr: 172.20.4.0/24
PrivSub2Cidr: 172.20.5.0/24
PrivSub3Cidr: 172.20.6.0/24

#Region Name
region: "us-east-2"

#Zone Names
zone1: us-east-2a
zone2: us-east-2b
zone3: us-east-2c

state: present

#Bastion Vars
bastion_ami: ami-083eed19fc801d7a4
MYIP: Your IP address/32
keyName: ansible-key
instanceType: t2.micro
```

### Step 2 - Ansible Setup
Create a new ```site.yml``` file
```
- import_playbook: vpc-setup.yml
- import_playbook: bastion-instance.yml
```

If you deleted your EC2 instance from the previous project, create another one
```
AMI: Ubuntu 20.04
Instance Type: t2.micro
Security Group: allow SSH on port 22
Create a new keypair
UserData:
#!/bin/bash
apt update
apt install ansible -y
```

SSH into the newly created instance and check the Ansible version
```
ansible --version
```

Install the AWS CLI on the instance
```
sudo apt install awscli -y
aws sts get-caller-identity
```

### Step 3 - Vars, AMI & Key Pairs
Create a new file in the ```vars``` folder called ```vprostacksetup```
```
nginx_ami: ami-06c4532923d4ba1ec
tomcat_ami: ami-06c4532923d4ba1ec
memcache_ami: ami-06c4532923d4ba1ec
rmq_ami: ami-06c4532923d4ba1ec
mysql_ami: ami-06c4532923d4ba1ec
```

Create a new playbook, ```vpro-ec2-stack.yml```
```
- name: Setup Vprofile Stack
  hosts: localhost
  connection: local
  gather_facts: no
  tasks:
    - name: Import VPC setup Variable
      include_vars: vars/vpc-output_vars

    - name: Import vprofile setup Variable
      include_vars: vars/vprostacksetup

    - name: Create vprofile ec2 key
      ec2_key:
        name: vprokey
        region: "{{region}}"
      register: vprokey_out

    - name: Save private key into file loginkey_vpro.pem
      copy:
        content: "{{vprokey_out.key.private_key}}"
        dest: "./loginkey_vpro.pem"
        mode: 0600
      when: vprokey_out.changed
```
Commit/push the changes to a new branch on the GitHub repository. Pull into the Ansible server and run the playbook
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/83047525-373b-44f1-969f-2c7bb5164150)

Run the ```vpro-ec2-stack.yml``` playbook
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/d14a9221-5edd-457b-8d7b-e09cb72e0bcf)

### Step 4 - Security Groups
Add new tasks to the ```vpro-ec2-stack.yml``` playbook to create security groups
```
- name: Create Securiry Group for Load Balancer
      ec2_group:
        name: vproELB-sg
        description: Allow port 80 from everywhere and all port within sg
        region: "{{region}}"
        vpc_id: "{{vpcid}}"
        rules:
          - proto: tcp
            from_port: 80
            to_port: 80
            cidr_ip: 0.0.0.0/0
      register: vproELBSG_out

    - name: Create Securiry Group for Vprofile Stack
      ec2_group:
        name: vproStack-sg
        description: Allow port 22 from everywhere and all port within sg
        region: "{{region}}"
        vpc_id: "{{vpcid}}"
        purge_rules: no
        rules:
          - proto: tcp
            from_port: 80
            to_port: 80
            group_id: "{{vproELBSG_out.group_id}}"

          - proto: tcp
            from_port: 22
            to_port: 22
            group_id: "{{BastionSGid}}"
      register: vproStackSG_out

    - name: Update Securiry Group with its own sg id
      ec2_group:
        name: vproStack-sg
        description: Allow port 22 from everywhere and all port within sg
        region: "{{region}}"
        vpc_id: "{{vpcid}}"
        purge_rules: no
        rules:
          - proto: all
            group_id: "{{vproStackSG_out.group_id}}"
```
Commit/push the changes to a new branch on the GitHub repository. Pull into the Ansible server and run the playbook
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/817233c6-13cf-4fee-bb87-5465154b3442)
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/e51f2206-a529-413c-971b-153954ddb2b0)

### Step 5 - EC2 Instances
Add tasks to the ```vpro-ec2-stack.yml``` playbook to create the EC2 instances
```
- name: Creating Nginx web01
      ec2:
        key_name: vprokey
        region: "{{region}}"
        instance_type: t2.micro
        image: "{{nginx_ami}}"
        wait: yes
        wait_timeout: 300
        instance_tags:
          Name: "web01"
          Project: Vprofile
          Owner: DevOps Team
        exact_count: 1
        count_tag:
          Name: "web01"
          Project: Vprofile
          Owner: DevOps Team
        group_id: "{{vproStackSG_out.group_id}}"
        vpc_subnet_id: "{{privsub1id}}"
      register: web01_out

    - name: Creating tomcat app01
      ec2:
        key_name: vprokey
        region: "{{region}}"
        instance_type: t2.micro
        image: "{{tomcat_ami}}"
        wait: yes
        wait_timeout: 300
        instance_tags:
          Name: "app01"
          Project: Vprofile
          Owner: DevOps Team
        exact_count: 1
        count_tag:
          Name: "app01"
          Project: Vprofile
          Owner: DevOps Team
        group_id: "{{vproStackSG_out.group_id}}"
        vpc_subnet_id: "{{privsub1id}}"
      register: app01_out

    - name: Creating memcache mc01
      ec2:
        key_name: vprokey
        region: "{{region}}"
        instance_type: t2.micro
        image: "{{memcache_ami}}"
        wait: yes
        wait_timeout: 300
        instance_tags:
          Name: "mc01"
          Project: Vprofile
          Owner: DevOps Team
        exact_count: 1
        count_tag:
          Name: "mc01"
          Project: Vprofile
          Owner: DevOps Team
        group_id: "{{vproStackSG_out.group_id}}"
        vpc_subnet_id: "{{privsub1id}}"
      register: mc01_out

    - name: Creating RabbitMQ rmq01
      ec2:
        key_name: vprokey
        region: "{{region}}"
        instance_type: t2.micro
        image: "{{rmq_ami}}"
        wait: yes
        wait_timeout: 300
        instance_tags:
          Name: "rmq01"
          Project: Vprofile
          Owner: DevOps Team
        exact_count: 1
        count_tag:
          Name: "rmq01"
          Project: Vprofile
          Owner: DevOps Team
        group_id: "{{vproStackSG_out.group_id}}"
        vpc_subnet_id: "{{privsub1id}}"
      register: rmq01_out

    - name: Creating Mysql db01
      ec2:
        key_name: vprokey
        region: "{{region}}"
        instance_type: t2.micro
        image: "{{mysql_ami}}"
        wait: yes
        wait_timeout: 300
        instance_tags:
          Name: "db01"
          Project: Vprofile
          Owner: DevOps Team
        exact_count: 1
        count_tag:
          Name: "db01"
          Project: Vprofile
          Owner: DevOps Team
        group_id: "{{vproStackSG_out.group_id}}"
        vpc_subnet_id: "{{privsub1id}}"
      register: db01_out

    - debug:
        var: db01_out.tagged_instances[0].id
```
Commit & push the changes to the GitHub repository. Pull into the Ansible server and run the playbook
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/1026b4a6-70dd-4b7e-bf25-b3f02145a324)

Confirm the instances were created
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/bf6ecf23-c3b1-4a46-84b7-c09119cb08ee)

### Step 6 - Load Balancer
Add task to create a load balancer
```
- local_action:
        module: ec2_elb_lb
        name: "vprofile-elb"
        region: "{{region}}"
        state: present
        instance_ids:
          - "{{ web01_out.tagged_instances[0].id }}"
        purge_instance_ids: true
        security_group_ids: "{{ vproELBSG_out.group_id }}"
        subnets:
          - "{{ pubsub1id }}"
          - "{{ pubsub2id }}"
          - "{{ pubsub3id }}"
        listeners:
          - protocol: http # options are http, https, ssl, tcp
            load_balancer_port: 80
            instance_port: 80
```
Commit & push the changes to the GitHub repository. Pull into the Ansible server and run the playbook
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/b9da95ad-1f88-4394-904f-3060e88caed7)

### Step 7 - Dynamic Inventory & Dynamic Varibales
Additional tasks to store the IP addresses of all the instances created

Create a ```hostsip``` file in ```provision-stack/group_vars``` folder <br>
Create a second file ```inventory-vpro``` in the ```provision-stack``` folder <br>
Add this to the ```vpro-ec2-stack.yml``` file
```
- name: Insert/Update Hosts IP & Name in file provision-stack/group_vars/hostsip
      blockinfile:
        path: provision-stack/group_vars/hostsip
        block: |
          web01_ip: {{ web01_out.tagged_instances[0].private_ip }}
          app01_ip: {{ app01_out.tagged_instances[0].private_ip }}
          rmq01_ip: {{ rmq01_out.tagged_instances[0].private_ip }}
          mc01_ip: {{ mc01_out.tagged_instances[0].private_ip }}
          db01_ip: {{ db01_out.tagged_instances[0].private_ip }}

    - name: Copy login key to provision_stack directory
      copy:
        src: loginkey_vpro.pem
        dest: provision-stack/loginkey_vpro.pem
        mode: '0400'

    - name: Insert/Update Inventory file provision-stack/inventory-vpro
      blockinfile:
        path: provision-stack/inventory-vpro
        block: |
          web01 ansible_host={{ web01_out.tagged_instances[0].private_ip }}
          app01 ansible_host={{ app01_out.tagged_instances[0].private_ip }}
          rmq01 ansible_host={{ rmq01_out.tagged_instances[0].private_ip }}
          mc01 ansible_host={{ mc01_out.tagged_instances[0].private_ip }}
          db01 ansible_host={{ db01_out.tagged_instances[0].private_ip }}
          cntl ansible_host=127.0.0.1 ansible_connection=local

          [websrvgrp]
          web01

          [appsrvgrp]
          app01

          [rmqsrvgrp]
          rmq01

          [mcsrvgrp]
          mc01

          [dbsrvgrp]
          db01

          [control]
          cntl

          [stack_inst:children]
          websrvgrp
          appsrvgrp
          rmqsrvgrp
          mcsrvgrp
          dbsrvgrp

          [stack_inst:vars]
          ansible_user=ubuntu
          ansible_ssh_private_key_file=loginkey_vpro.pem
          #ansible_python_interpreter=/usr/bin/python3
```
Check the content of the files
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/83fe28ee-035a-4bb3-8344-a4b29ed6c5c9)
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/3dc1a4af-3fc1-4868-8ad4-650cb4fdaecd)

### Step 7 - Wrap Up
Delete all the EC2 instances, NAT gateway
