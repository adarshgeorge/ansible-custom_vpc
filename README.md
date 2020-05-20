## AWS Custom VPC


Here the playbook creating.
* Custom VPC
* Two Public Subnet and one Private Subnet
* Internet Gateway
* NAT Gateway 
* Security group for Bastin, Webserver an Database
* Route Tables
* Launching instances with Webserver, Bastin Server and Database server
 
### Pre Request
 
An EC2 with full administrative access roles.  Ansilble, Python 3, boto and boto3 should be installed on this instance. 

**Launch Ansible Master EC2 instance**

Userdata
```

#!/bin/bash
sudo yum install python3 pyhton3-pip -y
sudo pip3 install ansible
sudo pip3 install boto3
sudo pip3 install boto

```


**Create an IAM Role with Policy AmazonEC2FullAccess and attach it to the Ansible master instance.**
```
AWS Console > Security, Identity, & Compliance > IAM > Access management > Roles > Create role

Select EC2 > Next: Permissions > Search "AmazonEC2FullAccess" > Add name and description > Create Role
```

Now we need attach the role to the created ansible EC2 instance.
```
EC2 > Ansible Server > Action > Instance Settings >  Attach/Replace IAM Role > Select roles "Ansible_Role" > Apply
```

**Ansible Playbook**

$ vim custom_vpc.yml

```
---
- name: Creating custom VPC . . .
  hosts: localhost
  vars:
    aws_region: us-east-2
    vpc_name: ansible_vpc
    vpc_cidr_block: 172.16.0.0/16
    public_subnet_1_cidr: 172.16.0.0/20
    public_subnet_2_cidr:  172.16.16.0/20
    private_subnet_1_cidr: 172.16.32.0/20
    keyname: Ohio_Home
    imageid: ami-0f7919c33c90f5b58
    instance_type: t2.micro
    webserver_exact_count: 1
    bastin_exact_count: 1
    database_exact_count: 1

  tasks:

    - name: Creating new VPC in AWS . . .
      ec2_vpc_net:
        name:       "{{ vpc_name }}"
        cidr_block: "{{ vpc_cidr_block }}"
        region:     "{{ aws_region }}"
        state:      "present"
      register: ansible_vpc_info


    - name: Creating VPC is in progress.....
      wait_for: timeout=10


    - name: Obtain & Assigning VPC ID into a variable
      set_fact:
        vpc_id: "{{ ansible_vpc_info.vpc.id }}"


    - name: Creating Public Subnet in Availability Zone a . . .
      ec2_vpc_subnet:
        state:  "present"
        vpc_id: "{{ vpc_id }}"
        cidr:   "{{ public_subnet_1_cidr }}"
        az:     "{{ aws_region }}a"
        region: "{{ aws_region }}"
        resource_tags:
          Name: "Public-Subnet-1"
      register: my_pub_subnet_az1


    - name: Retreive Public Subnet ID and save into variable
      set_fact:
        public_subnet_az1_id: "{{ my_pub_subnet_az1.subnet.id }}"


    - name: Creating Public Subnet in Availabilty Zone b . . .
      ec2_vpc_subnet:
        state:  "present"
        vpc_id: "{{ vpc_id }}"
        cidr:   "{{ public_subnet_2_cidr }}"
        az:     "{{ aws_region }}b"
        region: "{{ aws_region }}"
        resource_tags:
          Name: "Public-Subnet-2"
      register: my_pub_subnet_az2

    - name: Retriev Public Subnet ID and saving to in variable
      set_fact:
        public_subnet_az2_id: "{{ my_pub_subnet_az2.subnet.id }}"


    - name: Creating Private Subnet in Availabilty Zone c . . .
      ec2_vpc_subnet:
        state:  "present"
        vpc_id: "{{ vpc_id }}"
        cidr:   "{{ private_subnet_1_cidr }}"
        az:     "{{ aws_region }}c"
        region: "{{ aws_region }}"
        resource_tags:
          Name: "Private-Subnet-1"
      register: my_private_subnet_az3


    - name: Obtain Private Subnet ID and save into variable AZ-3
      set_fact:
        private_subnet_az3_id: "{{ my_private_subnet_az3.subnet.id }}"

    - name: Creating Internet Gateway and Attach to the VPC . . .
      ec2_vpc_igw:
        state: "present"
        region: "{{ aws_region }}"
        vpc_id: "{{ vpc_id }}"
        resource_tags:
          Name: "vpc_igw"
      register: my_vpc_igw

    - name: Retreive Internet Gateway ID and Saving to a variable..
      set_fact:
        igw_id: "{{ my_vpc_igw.gateway_id }}"


    - name: Creating a NAT gateway and assign Elastic IP, if a NAT gateway does not yet exist in the subnet.
      ec2_vpc_nat_gateway:
        state: present
        subnet_id: "{{ public_subnet_az2_id }}"
        wait: yes
        region:    "{{ aws_region }}"
        if_exist_do_not_create: true
      register: nat_gateway_az2

    - name: Set Nat Gateway ID in variable [AZ-2]
      set_fact:
        nat_gateway_az2_id: "{{ nat_gateway_az2.nat_gateway_id }}"

    - name: Wait for 5 Seconds
      wait_for: timeout=5


    - name: Creating public subnet route table . . .
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


    - name: Creating new private subnet route table in AZ-3 . . .
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


    - name: Creating security Group for Bastin Server . .
      ignore_errors: true.
      ec2_group:
        name: "BastinSG"
        description: "Bastion-sg"
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        rules:

          - proto: tcp
            from_port: 22
            to_port: 22
            cidr_ip: 0.0.0.0/0
        rules_egress:
          - proto: all
            cidr_ip: 0.0.0.0/0
        tags:
          Name: "BastinSG-Tag"

      register: bastin_sg
    - name: printing
      debug:
        msg: "{{ bastin_sg }}"


    - name: Obtain Bastin Security group id and saving into a variable
      set_fact:
        bastin_sg_id: "{{ bastin_sg.group_id }}"


    - name: Creating security Group for Webserver . . .
      ignore_errors: true
      ec2_group:
        name: "WebserverSG"
        state: "present"
        description: "Websrver-sg"
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        rules:
          - proto: tcp
            from_port: 80
            to_port: 80
            cidr_ip: 0.0.0.0/0
          - proto: tcp
            from_port: 443
            to_port: 443
            cidr_ip: 0.0.0.0/0
          - proto: tcp
            from_port: 22
            to_port: 22
            group_id: "{{ bastin_sg_id }}"


        rules_egress:
          - proto: all
            cidr_ip: 0.0.0.0/0
        tags:
          Name: "WebserverSG-Tag"
      register: webserver_sg


    - name: Obtain Webserver Security group id and saving into a variable
      set_fact:
        webserver_sg_id: "{{ webserver_sg.group_id }}"


    - name: Creating security Group for Database server . . .
      ignore_errors: true
      ec2_group:
        name: "DatabaseSG"
        state: "present"
        description: "Database-sg"
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        rules:
          - proto: tcp
            from_port: 3306
            to_port: 3306
            group_id: "{{ webserver_sg_id }}"
          - proto: tcp
            from_port: 22
            to_port: 22
            group_id: "{{ bastin_sg_id }}"
        rules_egress:
          - proto: all
            cidr_ip: 0.0.0.0/0
        tags:
          Name: "DatabaseSG-Tag"

    - name: Launching Web Server in the VPC . . .
      ec2:
        key_name: "{{ keyname }}"
        region: "{{ aws_region }}"
        instance_type: "{{ instance_type }}"
        image: "{{ imageid }}"
        group: WebserverSG
        vpc_subnet_id: "{{ public_subnet_az1_id }}"
        assign_public_ip: yes
        wait: yes
        count_tag:
          Name: WebServer
        instance_tags:
          Name: WebServer
        exact_count: "{{ webserver_exact_count }}"


    - name: Launching Bastin Server in the VPC . . .
      ec2:
        key_name: "{{ keyname }}"
        region: "{{ aws_region }}"
        instance_type: "{{ instance_type }}"
        image: "{{ imageid }}"
        group: BastinSG
        vpc_subnet_id: "{{ public_subnet_az2_id }}"
        assign_public_ip: yes
        wait: yes
        count_tag:
          Name: BastinServer
        instance_tags:
          Name: BastinServer
        exact_count: "{{ bastin_exact_count }}"


    - name: Launching Database Server in the VPC . . .
      ec2:
        key_name: "{{ keyname }}"
        region: "{{ aws_region }}"
        instance_type: "{{ instance_type }}"
        image: "{{ imageid }}"
        group: DatabaseSG
        vpc_subnet_id: "{{ private_subnet_az3_id }}"
        assign_public_ip: yes
        wait: yes
        count_tag:
          Name: DatabaseServer
        instance_tags:
          Name: DatabaseServer
        exact_count: "{{ database_exact_count }}"
```

#### Run the Playbook

```
$ ansible-playbook custom_vpc.yml 
```

## That's it!!!