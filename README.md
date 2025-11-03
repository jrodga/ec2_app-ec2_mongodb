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

```bash
sudo nano script_mongodb.sh
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
 
```bash
#!/bin/bash

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
     
**Make it executable and run it:**
```bash
chmod +x script_mongodb.sh
sudo install_mongodb.sh
```
**Note:**

Opening bindIp to 0.0.0.0 allows connections from any source permitted by Security Groups. Always restrict inbound 27017 in the Security Group to trusted sources only.

## 3) Create an AMI image

When MongoDB is installed and running, bake the image.

In the EC2 console, select the configured instance

Actions → Image and templates → Create image

Name: mongodb-ubuntu-22-04-v1

Reboot option enabled

Create image

After the AMI is available, you can launch future DB instances from this image. Apply a Security Group that allows inbound 27017 only from your App EC2 or a narrow CIDR.(i will suse 0.0.0.0 for testing not best practise)

Notes and housekeeping

Keep SSH closed once your image is baked and tested

For production, use EBS volumes with suitable size and IO, add CloudWatch metrics and backups

Consider pinning a specific MongoDB minor version in apt preferences if you need strict versioning


## App EC2 image build and launch guide

**This creates:**

- An App EC2 build script you run once to install Node, Nginx, PM2 and your app, then bake an AMI

- A User Data script to run on instances launched from that AMI that sets DB_HOST, seeds the DB and brings the app up

**1) Build the App EC2 and run the setup script**

**EC2 settings**

- AMI: Ubuntu

- Type: t3.micro 

- Security group: inbound HTTP 80 from the internet (i will suse 0.0.0.0 for testing not best practise)


**in SSH:**
```bash
ssh -i ~/.ssh/<your-key>.pem ubuntu@<APP-EC2-Public-DNS>
```

**Create the build script:**

```bash
sudo nano app_script.sh

```
**inside the document**


