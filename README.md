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
* Create EC2 instance for Private Subnets. Add below script for weblogic server. the `-y` is to answer all questions as Yes.
```shell
#!/bin/bash
yum update -y
yum install -y httpd
service httpd start
```
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

`nslookup myip.opendns.com` `resolver1.opendns.com` look for non-authoriative IP

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
* CORS basic setup

Allowing contents from bucket1 in bucket 2, this is setup in bucket2 cors to allow bucket1 file access
```html
<?xml version="1.0" encoding="UTF-8"?>
<CORSConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
<CORSRule>
    <AllowedOrigin>http://cfst-17-6b79369d3f6aac17fc601c9ac00b909-s3bucket1-6gxq45m57poh.s3-website-us-east-1.amazonaws.com</AllowedOrigin>
    <AllowedMethod>GET</AllowedMethod>
    <AllowedHeader>Authorization</AllowedHeader>
</CORSRule>
</CORSConfiguration>
```

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

# Route 53

* It is a DNS Hosting Service. It has
    * Domain Registration: Getting a url
    * Domain Name Registration(DNS) Service: url to ip adress, this is done by using Name servers (NS)
    * Health Checking

## Hosted Zones

* Contains DNS rules(record sets) as in what to do with the web traffic coming in. 
* It also contains Name Servers

## Record Sets

* Instructions matching Domain Name to IP
    * A: IPv4
    * AAAA: IPv6
    * CNAME: point a name to another name
    * MX: route email (mail server)
* Alias Record set
    * Points to specific AWS service eg: ELB, EBS, S3 etc
    * This is type A
    * This is because directing it direct IP, IP may change for AWS services
* Routing Policy
    * Simple: Routing all traffic to one end point
    * Weighted: Manual Load Balancing
    * Latency: Route based on users latency to various end points
    * Failover: Route traffic to secondary if primary is not available
    * Geolocation: Route based on location

## DNS Failover

* Keep the ELB/S3, the primary source as primary. In the failover
* Health check should be enabled at primary for this feature to work
* For the S3 or the secondary, set as failover and keep secondary. 

# Steps to Host

* Buy a Domain Name from a Domain Registrar like GoDaddy or from AWS or from free services
* Create a Hosted Zone with the Domain you got. This will create Name Servers, put these Name Servers in the Domain Name Name Servers at the Registrar's website
* Create a record set pointing to the AWS service you want to use.

# Steps in Traffic routing: Simple Version

* From your browser you type google.com
* Request first checks browser cache if not it  goes to your ISP (eg Cox) DNS Resolver 
* if not there it goes to Root Name Server
* if not root name server tells it to check .com Top Level Domain (TLD) server
* if not .com server tells the name of name server to check
* here it interacts with route 53 which has the name servers, and the name servers gives the ip address for google.com
* based on the alias set this is the aws service which then routes the traffic to the necessary server and send the information you requested on google.com page

# Cloud Front

* You have a static web page being hosted from S3
* You can directly use the url to share the website given by s3, without using Route 53 or Cloud Front. The problem is the url is unreadable so you want easy to read url. To address this problem, you use route 53. The problem is you are hitting your s3 again and again and s3 is located at one place and users are all over the world, to address this problem you use cloudfront distribution.
* You can combine the above two. In Cloud Front you tell the origin which is the S3 bucket and it will give you a domain name, which you can use to access S3. Or you can configure the Cloud Front to use an alternate DOmain Name(CNAME). Then in Route 53 create a new record set, and set an alias pointing to the Cloud Front. This way you can use the domain name you want.
* Now you can use a custom domain name as you want, and also server traffice with low latency thanks to cloud front. 
* Data transfer from S3 to cloudFront is free
* It is a global CDN (Content Delivery network). It basically is a proxy server(s) which caches traffic from the original server and then traffic is routed to this proxy server instead of original server, hence reducing load.
* The original server which is ELB or S3 is called Origin and proxy server is called Edge Location
* Route 53 is used where a Record set with CNAME is created routing traffic for the domain to another domain which can be like routing meditations22.com to cdn.meditations22.com
* Even if TTL for an object is 0, it's faster in Cloud Front as it replaces only if there is a change in header version, so if it's not it doesn't replace and serves the same object
* You can restrict the access to Origin S3 in Cloud Front so that users don't have direct access to it.


# VPC Peering

* To ping server we need to have All ICMP- IPv4 type protocol ICMP(1) open. To ping it from anywhere use `0.0.0.0/0` and to ping from paritcular VPC use the ip pf that, if more than one vpc, use a top level ip block. For example, `10.0.0.0/13` covers from `10.1.0.0` to `10.7.255.255`
* Create a VPC peering connection by giving a requestor VPC and acceptor VPC
* After creating a peering connection, go to the acceptor VPC account, and click accept. Till then it will be pending
* After creating just VPC peering we still won't be able to ping.We need one more step, add traffic routing to route table. 
* Route needs to be added on both sides of the relationship. You give the other VPC IP as Destination and Peering connection ID as Target. 
* Once VPC is established you can only ping personal IP, not public IP, as for public IP you need to go outsider internet and we removed the open internal ping rule from NACL by limiting to only VPC
    * To work around this, go to vpc peering connection
    * Actions-->Edit DNS settings
    * DNS Resolutions allow for both VPC
    * This will work only for public DNS not Public IP 
* We can connect private subnets as well, until and unless the route table has peering connection route
* Three steps for VPC peering
    * NACL, ICMP 1 open for VPC IP
    * VPC peering connection established, accepted and working
    * Route table VPCE source and Peering ID as destination in each subnet
    * In Peering go to Action and Edit DNS Setting, enable.

# Database

RDS is fully managed.

When you create a Primary DB, you can create a Read Replica, which is asynchronous copy of primary DB

## Neptune
* Full managed NoSQL Graph DB, different from DynamoDb as DynamoDb is Document Store/Key-value store. Neptune is similar to Neo4j and DynamoDB is similar to MongoDB
* Open Source API
    * TinkerPop Gremlin - Uses Gremlin Console Client
    * RDF SPARQL - Uses RDF4j Console
* Highly Available
* Storage up to 64 TB, up to 15 Read Replicas
* Server Side Encryption

# SNS--Simple Notification Service

* There are many services for which we want to receive alert. Like S3 upload, delete etc. We can't set up alerts from the service directly. For this we have SNS.
* When we set alert we need to give where the alert should go, mostly it is email it could also be other things. You can't just go to SNS and set an email as you need a place to define who is sending the alert and a place to group all the emails. So you basically crate such group to contain all rules, this is called SNS Topic
* Instead of policy you can create a role to use the sns service and attach that role to the s3 or lambda function. This way you don't have to give any policy rule at SNS level
* Once we create SNS topic, we will put in all the emails which are called subscribers, there can be other subscribers too like Lambda function, SQS and text
* That is not enough, as we need to tell the group who will be sedning notification for this we need to go and edit the policy and add the required bucket name or whatever service. 
* In the service, that is S3 we will have to setup the events to go to the SNs topic created. 
* If email, we will have to confirm the email. That is it. 
* SNS Work Flow
    * Publisher - Sender. Eg: Human, S3 event, Cloud Watch Alarm
    * Topic - Group which contain subscribers, IAM rules etc. 
    * Subscriber - Receiver : Eg SMS, Email, Lambda, SQS
  
# SQS - Simple Queue Service

* This is used to create a decoupled architecture
* Messages between servers are retrieved through polling
    * Long Polling--1-20 seconds, Less API calls so cheaper
    * Short Polling--More calls, so costlier
* Mangaed by AWS, Highly Available
* Msg can contain 256 KB, text format
* Standard Queue and First in First Out (FIFO) Queue

# SWF - Simple Work Flow

* Used to create distributed application
* Workflow - Sequence of steps required to perform a task
* Worker - Responsible for performing a task
* Task: What work needs to be done
* Activites: A single step in the workflow

# API Gateways

* Allows you to create and manage your own API for your application
* RESTful API with Resources, Methods and Settings
* Different stages, DEV, TST and PRD
* Roll back to previous deployment, versioning done
* Custom Domain names
* Monitor using CloudWatch for Keys, erros, caching and latency
* API caching is done, so that API response to a duplicate API doesn't hit back end
* Can be used with cloud Front to reduce latency and improve security


# Lambda Function

* You can create a Lambda Function using many platforms Python, NodeJS, Go

# Lambda - Excercise

* Set up Lambda with CloudWatch and SNS IAM roles
* Create a Lambda Python program to access sns topic and send a message
* Create a Cloud Watch rule to schedule sending message for x days or x minutes
* in SNS topic, create a topic, add subscribers, use the TopicArn in the Lamda python code
* Create a Lambda function with the code and add a IAM role with permission for SNS and Cloud WAtch. This way you don't have to create any policy at SNS and cloudWatch level
* Start the Lambda and we will have a serverless function which is sending messages at a given frequency to a list of subscribers. 

# Deployment -- Cloud Formation

* Infrastructure as code
* Create a simple template, to define the infrastructe and Cloud Formation will create the resources for you, called stack
* Templates create a stack, we use a single template to create multiple stack. 
* Template is in JSON or YAML format
* Versioning support. You can roll back to previous version if current fails
* Good for DR, as we don't need a backup copy running all the time, we just need a cloud formation template from which we can create a stack and it will be up in no time. 
* Supports Nested Stacks
    * Different template for Ec2, IAM, DB and have templates calling other templates.   
    * Hence having micro-templates instead of a single monolith one.
  
## Template

* Resources 
    * Services to deploy VPC, EC2, IGW etc
    * User Data Scripts
    * Custom Resources
* Parameters
    * Variables in templates
    * User prompted for values at run time
    * Key pairs, instance types, DB password etc
* Mapping  
    * Sometimes we need names, like AMI name or subnet name. To avoid going back and checking we create Look up tables, which is stored in the template itself from which we can copy the value and use 
* Output
    * Information to receive from stack. Like ELB DNS name
* Conditions
    * If Env = PRD then Auto Group = Yes

* Delete Stack will delete all the resources created via Cloud formation

# Monitoring

## Cloud Watch

* Monitoring all AWS services, EC2, S3, ELB
* We configure Cloud Watch Metrics and based on that monitor the Env
* Detailed vs Basic, based on plan chosen, cost varies
* Can create alerts based on thresholds set using SNS or create actions to other AWS services
* Cloud Watch is the basic pillar in Auto Scaling

## Cloud Trail

* API logging service, logs ALL API calls. 
* All logs are placed in the destinated s3 bucket

## VPC Flow Logs

* Logs incoming IP traffic to VPC
* Logs stored in CloudWatch
* Can be created for specific subnet
* To create flow log we need IAM role givnig access to CloudWatch to log the event
* Flow log map
  
```  
173.244... - Source IP
10.0.2.32 - Destination IP
623628 - Source Port
22- Destination port
6- protocol number
28 - packets transfered
4237 - bytes
time at the start of capture
time at the end of capture
status accept or reject
status of log
```
## AWS Config

* Visibility into resources and configuration. Using AWS Config you can see what resources are running on your AWS account
* Relation between resources: As in which resources use a particular SG
* Historical timelines
* Integration with CloudTrail
* Notifications: When resources are created, modified or deleted
* Config Rules:
    * Evaluates Compliance: Internal rule that all S3 data must be encrypted
    * Managed Rules, provided by AWS, which can be used
    * We can also create a custom rule, by creating a Lambda function for that rule

## Lab

* Create a S3 bucket
* Create a Cloud Trail for that S3 bucket and store the logs in another bucket
* Create a Cloud Watch for the Logs, by creating an IAM role with Access to write the logs assigned to CloudTrail. You don't have to go to IAM page for this
* Create a Cloud Metric by Creating an Alarm. --read documentation on Cloud Alarm
* Send Notification via SNS

The Cloud Trail doesn't log at the same time, there is a lag, around 15-20 minutes
 
# ECS - Elastic Container Service
* Container Management Service like Docker. This is useful when you have many containers
* Components
    * Task Definition: Point in time capture of the configuration of running the image. where as Docker Image is the point in time capture of the running applicaiton
        * Docker Images for each container
        * CPU and memory requirments for each container
        * Links between container
        * Networking and Port setting
        * Data Storage Volume
        * Security IAM roles
    * ECS Clusters
        * Group of EC2 instances, each running ECS
        * ECS Agent is one light EC2 instances giving direction to other EC2 instances
    * ECR - Elastic Container Registry
        * Store Manage and Deploy Container Images
        * Fully Managed Docker Container Registry


# EBS - Elastic Bean Stalk

* Automated Deployment and Scaling Service
    * Load Balancing, Auto Scaling, Monitoring, Platform Management and Code Deployment
* Deployment Automation
    * In-Place (Upload, and Deploy)
        * Rolling update: Not updating each environment. Good for minor updates
    * Blue-Green 
        * Helps you run different version in different stages   
* When creating a EBS application, make sure a VPC is present

# Analytics

## Kinesis

* Real time processing of Data
* Data generated from producers, goes to Kinesis, which generates multiple streams where consumers like DynamoDB, S3, Kinesis App consume the data, without data loss.
* Kinesis Video Stream: Real time video processing
* Kinesis Data Stream: 
    * Streams: Contains 1 or more shards
    * Shards: 
        * This is processing power. 
        * 1 MB/sec read, 2 MB/sec write. 
        * Billed by number of shard hours
    * Producers: Mobile, IoT devices, install agents on servers to send log file live
    * Consumers
* Kinesis Firehose: Load streaming data to S3, Splunk, RedShift, Elastic search
* Kinesis Data Analytics: Run SQL agianst Data Stream of Firehose, send out to Data Stream or firehose

## Elastic Map Reduce (EMR)

* Solve the problem of data created in High variety, High volume and High velocity. This was done by creating frameworks(Hadoop, Spark) for distributing work to mulitiple servers.
* EMR runs this framework(Hadoop) on EC2 instances
* Data is first stored in S3 and from there the EMR takes it for processing and the output can be stored in S3 or somewhere else. While processing the dat is stored either in HDFS (Hadoop File Sytsem) or an AWS product which utilizes S3 is called (EMRFS)
* Master Node: Controls Slave Node, This is the only Ec2 instance we have access to
* Slave Node:
    * Core Node: Processes and stores data. Mandatory to have at least one core node
    * Task Node: Only Process
* Map-Reduce: In Map, data is split to slave nodes. In Reduce, data is aggregated into a single file and output stored

# AWS Well Architected Framework

A well architect framework has following traits

* Operations Excellence: Runs and monitors systems to provide business value and continually improve supporting processes and procedures
* Reliability: Recover from failure and mitigate disruptions
* Security: Protect and Monitor System
* Performance Efficiency: Use computing resources efficiently
* Cost Optimization: Avoid or eliminate unneeded cost

## Operation: Best Practices and Key Services

### Design Principles
* Infrastructure/Operations as Code: Automate everything Eg: Auto Scaling
* Annotate Documentation: Code everything, which will serve as documentation
* Frequent, small, reversible changes--Monitor test and quick feedback. This way it is less risky.
* Refine Operation procedures continiously
* Anticipate failure: Assume whatever can fail, will fail. In place operations to correct when component fails. Like in Auto scaling we have a monitor setup to replace failed instances with healthy one
* Learn from all opearation failures: Why it took so much time to recover, why did we have data loss

### Key services
* Operations as a Code: Cloud-Formation
* Log Collection and Monitoring: Cloud Watch, Cloud Trail, VPC Flow logs, AWS Config
* Have procedures to follow when things break, and schedule game days to make the team practice fire drills.

### AWS DevOps Tools

* CodeCommit: Managed git repo, similar to git. When code commited to CodeCommit it can trigger CodeBuild
* CodeBuild: Build and test the application code before releasing
* CodeDeploy: Deploy code to Ec2 instances, can do rolling updates
* CodePipeline: Can orchestrate all the above to give the CI/CD
* CodeStar: Complete Project Management Solution, that includes things like Bug Tracking and CI
* Xray: Optimize service oriented architecture


## Reliability

The ability to recover from failure and mitigate disruptions

### Design Principles

* Test recovery procedure: To see what happens when x goes down, and make sure we have acceptable level of recovery based on time and data lost. 
* Automate recovery: Whenever possible, automate recovery.
* Scale Horizontally: Most possible reason for failure is increase in traffic
* Stop guessing capacity: Change mindset, automate peak load
* Automate change: Follow DevOps Principles, like small changes, if problem roll back, stop manual changes use scripts like cloud formation templates

### Key Services

* Cloud Watch
* Foundations: 
    * IAM: limit only authorized people to make changes
    * VPC: Different environments for DEV, TST and PRD
    * Trusted advisor: Service limits of EC2 and EBS, so that you can pro-actively contact AWS and increase limits such that we are not stuck with system not able to auto scale because service limit is reached.  It can send weekly reports. Apart from this it also provides other services like
        * Service Limits
        * Cost Optimization (Idle resources, capacity reservations)
        * Performance (High utilization)
        * Security (IAM Permissions, SG)
        * Fault Tolerance (EBS Snapshot, RDS backup, Multi A-Z)
    * DDOS Protection : Sheild standard everyone gets. Advance WAF (Web application firewall) allowing to firewall at application layer
* Change Management: 
    * Cloud Watch: Control Access
    * AWS Config: Configuration Awarness
    * CloudTrail: Audit API calls
    * AutoScaling: Demand Management
* Failure Management:
    * Cloud Fromation: Infrastructure is completely scripted
    * S3: Durable backups, to mitigate against corruption or deletion of data
    * KMS: Reliable key management service, to make sure we don't lose access to encryption key

### Encryption on AWS

* They key management system has a service to
    * Generate keys to encrypt
    * Encrypt the data using those keys and generate encrypted data
    * encrypt the plane key to generate the key used for encyption
    * store the encrypted data and encrypted key
    * When we need to decrpt, decrypt the encrypted key and use that key to decrpyt the data
* AWS KMS: Generates the Keys and encrypts plane key to encrypted key and Decrypts. Integrate with EBS, S3, RedShift, Kinesis, EMR, SQS, Workmail, transcoder
* CloudHSM: KMS uses hardware not dedicated to just you, this may violate some security compliance requirments.For this we use CloudHSM. When we need dedicated Hardware. Not as integrated as KMS with AWS services.

### Disaster Recovery Design Patterns

* RTO: Recovery time objective- How long to recover
* RPO: Recovery Point Object- How much data can be lost

Popular Deisgn Patterns are

* Backup and Restore
    * Backup data to AWS second region (s3, snapshot) based on RPO
    * Have AMIs in recovery regions
    * Cloud Formation template standing by
    * During Disaster
        * Spin up instances from AMI using template
        * REstore backup dat
        * Modify DNS to point to new instances using Route53
* Pilot Light   
    * Cross region replication: RDS, DynamoDB, S3
    * Have ready stopped instances
    * Smaller DB instances
    * During Disaster
        * Start Instances
        * Scale up DB, promote to primary
        * Modify DNS or Use Route 53 to failover
* Low Capacity Standby  
    * Similar to Pilot Light but not have all instances running
    * Some capacity running 24/7
    * Complete replication of DB, multiple Masters, by using Aurora
    * During Disaster:
        * Scale Up/ Auto scale
        * Route 53 failover to DNS
* Multi-Site Active Active
    * Two regions replicated. 
    * During Disaster:
        * Route 53 failover to DNS


## Security

The ability to protect informations, systems and assests while delivering business value through risk assessments and mitigation stratergies

### Design Principles
* IAM: 
    * Implement a strong Identity foundation
    * Principle of least privilege
    * Root access disable
    * AWS Organizations: Centrally Manage accounts
* Enable Tracability:
    * Cloud Trail to see API Calls
* Apply Security at every Layer:
    * Security Group
    * NACL
* Automate Security:
    * Cloud Formation Template
    * AMIs with proper security are in that templates, make sure no one can launch any other AMI
* Protect data in transit and in rest
    * Encryption
* Prepare for Security Events:
    * Lambda function tried to events that initate at any breaches

### Key Services

#### IAM

* IAM
* AWS Organizations: Centrally Manage accounts
* MFA: Hardware of Virtual Multi Factor Authentication, Console and API
* STS Temporary Security Credentials

#### Detective Controls

* AWS CloudTrail: API access Logs
* AWS Config: Resource Inventory
* AWS CloudWAtch: Logs Metric Filters
* Guard Duty: 
    * Intelligent Threat Detection
    * Machine Learning based getting data from
        * CloudTrail Events
        * VPC Flow Logs
        * DNS Logs
        * Rules will be created in CloudWatch, which can trigger Lambda function, like blocking an IP address
  
#### Infrastructure Protection

* VPC: Isolated Virutal Networks
* Amazon Inspector: Vulnerability Detection
* AWS Shield: DDOS Mitigation
* AWS WAF: Application Firewall
    * SQL injection
    * Cross Site Scripting
    * Bad Bots
    * High Request Rates
    * Large REquest Size
    * Blacklist/ Whitelist IPs

#### Data Protection

* Amazon Macie
    * ML based Data Classification
    * Data stored in S3
    * Risky or suspicious activity alerted
        * Someone downloading data or security check for buckets
    * Alerts
        * High Risk Data Events
        * Credentials in source code
        * Uncrypted backups
* Encryption:
    * S3 object encryption
    * EBS block encryption
* KMS


## Performance Efficiency

The ability to use computing resources efficiently to meet system requirments and to maintain that efficiency as demand changes and technologies evolve

### Design Pattern

* Demorcatize advanced tech: Don't re-invent the wheel. If there is AWS service, just use it
* Go Global in minutes: Use Cloud Front
* Use Serverless architecture: Managed by AWS, less costly
* Experiment more often
* Mechanical sympathy: Understand AWS services

### Key Services

#### Selection

* Compute: Auto Scaling
* Storage: EBS, S3 for storing static content and serving from there
* DB: DynamoDB, RDS
* Network: VPC, Route 53, Direct Connect

#### Monitoring

* Metrics, Alarms and Notifications: CloudWAtch
* Automate Actions: Lambda

## Cost Optimization

Ability to avoid or eliminate unneeded cost or suboptimal resources

### Design Principles

* Adopt a consumption--cloud--model instead of on premise model
* Meansure overall efficiency using cloud watch
* Stop spending money on data centers
* analyze and attribute expenditure
    * Who is running what, do this based on add ing tags
* Cost effective resources
    * right instances and storage options
* Matching supply and demand
    * Auto Scaling
* Expenditure awareness
    * Cost allocation tags
    * Cloud Watch alarms for budge
* Optimizing over time
    * Contiously evaluate
    * Trusted Advisor


# Labs
## Simple Route 53 EC2 to S3 failover lab

* Start EC2 instance, use following commands to setup Apache with a webserver

```shell
sudo su
yum update -y
yum install httpd service
service httpd start
chkconfig httpd on # to start the service after reboot
cd /var/www/html
vim index.html
<html><h1>This is the main site</h1></html>
```
* go to route 53, create a hosted zone
* create a health check
* give the hosted zone and EC2 ip
    * remember to change IP here when we change reboot server if it's not having elastic IP
* keep health check threshold 1 so that there is no lag when the site fails and back up comes fast
* create a record set, with no alias and give the ec2 ip address
* create a s3bucket with hosted zone name (no period)
* upload a index html file, give public access
* make the statick website available, in the index give the index file name which you upload
* go back to route 53 and create a new record set, failover--secondary, alias yes, give s3--may have to refresh the entire page if s3 not visible.
* bring the ec2 down to see if the setup works. 
    * When you bring ec2 up agian, the ip may change, make sure you changeg both the hosted zone and health check IP.


ECS Lab Work from AWS Documentation