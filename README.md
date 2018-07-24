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



# user-data commands

`curl http://169.254.168.254/latest/user-data/`

`curl http://169.254.169.254/latest/meta-data/`

