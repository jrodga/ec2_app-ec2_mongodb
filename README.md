# MongoDB on EC2 with image bake

This guide shows how to:
1) Launch a base Ubuntu EC2 instance
2) Create and run a MongoDB install script on the instance
3) Bake an Amazon Machine Image from that configured instance

## 1) Launch the EC2 (base host)

- AMI: Ubuntu 22.04 LTS Jammy
- Instance type: t3.micro or similar
- Security Group for the DB image:
  - Inbound 22 from your IP for setup (i will suse 0.0.0.0 for testing not best practise)
  - Inbound 27017 from your App EC2 security group or VPC CIDR only (i will suse 0.0.0.0 for testing not best practise)
- Key pair: create or select one
- Storage: default is fine for a lab

Once running, connect:
```bash
ssh -i <your-key>.pem ubuntu@<EC2-Public-DNS>
```

## 2) Create the MongoDB setup script on the EC2

```
sudo nano ~/install_mongodb.sh
```

-  inside the document we create all the nesessary comands
    - Update Ubuntu
    - Upgrade Ubuntu
    - Install dependencies
    - Import mongodb public key (we are getting specifc version )
    - we update again to match packeges
    - we need to enable (will runing from the boot of th ec2)
    - start mongod
    - chanhge the Bin Ip
    - restart mongod
    - check the status

   - Put the next comndas in the script
 
```#!/bin/bash

# update and upgrade the ec2 

echo " Updating system packages..."
sudo apt update -y
sudo apt upgrade -y

echo "Installing dependencies..."
sudo apt install -y gnupg curl

# import mongodb public key

echo "Importing MongoDB public key..."

curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

#update the system 
echo " Updating system packages..."
sudo apt update -y

# get an espeifc version of mongo db

sudo apt install -y mongodb-org

echo "Enabling MongoDB service..."
sudo systemctl enable mongod
sudo systemctl start mongod

#change the local host to wild cart (0.0.0.0)

sudo sed -i 's/bindIp: 127.0.0.1/bindIp: 0.0.0.0/' /etc/mongod.conf

# re-start mongo 

sudo systemctl restart mongod

#chack status 

sudo systemctl status mongod
```
     

  
