# Security Group vs NACL

* Security Group is stateful where as NACL is not. That means there is no need for outbound traffic in Security Group, it will remember
* Security Group is at instance level where as NACL is at subnet level
* Security Group has no deny rule, only allow. 
* NACL is not mandatory, it is one extra layer of security
* Ephemeral ports are used for ssh(80)/rdp(3389) outbound in NACL 1024-65535

# EC2 Components

* Amazon Machine Image (AMI): OS and other settings--user data scripts
    * User Created
    * Community - Open Source
    * Market - Paid
* Instance type: Hardware (RAM, network bandwidtch etc)
* Network interface: Public, private or Elastic IP
* Storage: Hard drive
    * EBS-Elastic Block Store--Network drive, persistent storage. 
        * Steps to mount as File System drive
            * `lsblk`
            * `sudo mke2fs /dev/<ebsname>`
            * `sudo mount /dev/<ebsname> /mnt`
        * CAN NOT be accessed by more than one EC2 instance
    * Instance Store- ephemeral storage--temporary
    * EFS-Elastic File System -- 
        * Scalable storage option, no need to pre-allocate
        * Fully managed by AWS
        * Can be accessed by more than one EC2 instance
        * can be used as hybrid by mounting to on premise server when connected to AWS direct connect
        * The ec2 instances should have the default security group of the VPC added else EFS won't work



# user-data and meta-data

User data are custom boot strap commands which we enter during the initiation of an instance

meta-data displays instances meta data, AMI instance type etc

`curl http://169.254.168.254/latest/user-data/`

`curl http://169.254.169.254/latest/meta-data/`

# Elastic Load Balance and Auto Scaling

## ELB

* ELB is for distributing traffic across instances and Auto Scaling is grouping of instances in such a way that the group increases the number of instances with the increase in traffic
* ELB has its own DNS record set hence direct access. This can also be used internally for private subnets
* You can apply SSL directly to ELB

## Application ELB

* Can Load balance based on 
    * Path based rules: xyz.com/videos will go to videos group and xyz.com/pics will go to pics group
    * host based rules: a host is the fourth  part in a http request. When we call web services we use METHOD, RESOURCE, PROTOCOL VERSION AND HOST HEADER. example below. The other way to look at it is domain name, i.e xyz.com and abc.com pointing to same ELB but going to different servers.
```
GET xyz.html HTTP\1.1
Host: journal.com
```
## Auto Scaling

Has three parts

* Launch Configuration
    * This is the instance template which has all the details to be used while scaling (eg: AMI, Security Group, user-dat, storage etc)
* Auto Scaling Group
    * Rules that govern if/when an instance is automatically provisioned
        * No. of Min/Max instances 
        * VPC and AZ instance should be launched in
        * SNS notification
* Auto Scaling Policy
    * Auto scale increases/decreases instances based on *Cloud Watch* metrics


# Bastion Host

* This is EC2 instance in public subnet used to access instances in private subnet in the same VPC

# NAT Gateway

* Similar to Bastion but reversed, where the private instances use the gateway to download something from the internet
* They should be part of the private subnet route table. This is how the connection is established

# Lab excercise

* Create VPC
* Create IGW, attach to VPC
    * Only 1 IGW to a VPC, demonstrate
* Crate Subnet
    * For ELB to server traffic to instances in private subnet, it should be in public subnet of the same AZs. So if we have two subnets in two AZs then we need to first create two public Subnets in that AZ and attach to the ELB
* Create NACL, NACL is firewall at the subnet level
* Create 2 Route Table one is public and the other is private. 
    * Add IGW to Route Table Public
    * Add Subnets as required
* Create EC2 instance for Private Subnets
* Explain why we can't log in
* Create a Bastion Host, same EC2 but in Public Subnet
* Once Bastion Host is created, SSH into it and from there SSH into the Ec2 instances in the Private Subnet.
    * Open PuttyGen, Load the pem key and save without passphrase with the same name this will generate the ppk key, which we will use with putty
    * open putty add the Bastion host public IP, go to SSH-Auth, browse, give the location of the ppk key. Go back to session and Open
    * Show permission denied when ssh to webserver, because key is not added. 
* Adding private key in windows, using ssh tunneling
    * Go to Putty app folder, and open pageant
    * add the ppk key
    * open putty, in auth don't browse the key and select forwarding
    * once in bastion host, ssh to private server
    * *NOTE* If NACL is not ALL OPEN, the bastion host inbound ephemeral should also be open 
* Create a NAT Gateway, this is in VPC Console not EC2
    * NAT gateway use elastic IP
* In the Private Route table which has the private subnet where the webservers are located, add the route to NAT gateway, 0.0.0.0/0
* When Create ELB, we can't select private subnets. So we select the public subnet which are in the same zone as private subnet. Once we select that, it will show up the instances which are in the private subnet as well. 

# Security NACL

To check your personal public IP go online and check on website what is my ip ipv4 or in cmd type

`nslookup myip.opendns.com resolver1.opendns.com` look for non-authoriative IP

# Transfering files

Transfer first to bastion host and from there to the private server.

To transfer to/from windows 

* Start Pageant agent, which comes with putty. Add private ppk key to that and let it run.
* use a `pscp` this comes with putty. To transfer from server to windows go to cmd go to pscp location and type the following command

`pscp ec2-user@34.205.225.115:/home/ec2-user/* C:\Users\asyed\Downloads\*`

# S3

## S3 Componenets

* S3 Bucket
    * Main storage container.
    * Similar to folder/ website url. 
    * Name needs to be unique
* S3 Object
    * Individual file
    * 0 bits size to 5 TB
    * Server Side Encrytion (SSE) AES-256

## S3 Permissions
* Can be given at IAM or to bucket level
* IAM- Give s3 full access, then the user can see any bucket in S3
* Bucket policy-- attached to the bucket and not user
    * example. https://docs.aws.amazon.com/AmazonS3/latest/dev/example-bucket-policies.html
* S3 ACL--Granted for other AWS accounts, present at object and bucket level
    * Using object ACL we can share a file with public

## S3 Storage Classes
* Use LifeCycle Policies to automate this, can't upload directly in glacier
* Uploads
    * Single Upload, upto 5Gb, recommended 100 Mb
    * Multi-Upload upto 5 TB
    * AWS Import/Export upto 16 TB
    * Snowball > TB--Petabyte
* Used for cost efficency, Durability and Availability
* Standard
    * 99.999999999 (eleven 9s)--Durability--Very durable
    * 99.99 Availability -- always avialable
    * Most expensive
* Reduced Redundancy Storage - RSS
    * 99.99% Durable
    * 99.9% Available
    * Same availability but less durability, cheaper
* Infrequent Access (S3-IA)
    * 99.999999999% Durable
    * 99.90 Available
    * Same Druability, less availability
* Glacier
    * Bulk--5-12 hours to extract data
    * Standard--3-5 hours
    * Expedited--1-5 minutes
  
## S3 Pricing

Normally if you use a storage service you get charged by storage size. But S3 charges are different you are being charged by

* Storage Size
* Storage Type
* Storage API use Count
    * GET
    * PUT
    * DELETE
    * LIST
* Data Transfer Charge
* Versioning
* Event Notification

## Features

* Versioning
* Static Web hosting
    * Create a Bucket, enable static hosting, give index.html and error.html files, no need to give folders, as it's flat storage folder are for us to organize
    * Make files public
    * Use the link to view
    * CORS-Cross Online Resource Sharing
* Website down, backup

# TODO

Route 53
VPC Peering