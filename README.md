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
sudo source app_script.sh
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

```bash
#!/bin/bash

# Update package lists and upgrade installed packages
# Keeps the base image current and patched
sudo apt update -y
sudo apt upgrade -y

# Install core tools we need:
# - git to clone the app repo
# - nginx as the reverse proxy for the Node app
# - curl to fetch the NodeSource setup script
sudo apt install -y git nginx curl

# Install Node.js 20 LTS from NodeSource
# This adds the Node 20 repo to apt, then installs nodejs (includes npm)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo bash -
sudo apt install -y nodejs

# Quick check (optional) to see versions
node -v
npm -v

# Move into the ubuntu home and clone the app repository
# If the folder already exists (re-running the script), 'git clone' will fail, which is fine for a simple build step
cd /home/ubuntu
git clone https://github.com/LSF970/se-sparta-test-app.git

# Install app dependencies inside the app folder
cd /home/ubuntu/se-sparta-test-app/app
sudo npm install

# Install PM2 globally (process manager for Node apps)
# Then stop any previous PM2 processes (harmless if none), start the app, and save the PM2 process list
sudo npm install -g pm2
pm2 kill || true
pm2 start app.js --name sparta-app
pm2 save

# Configure PM2 to start on boot for the 'ubuntu' user so the app survives reboots
sudo pm2 startup systemd -u ubuntu --hp /home/ubuntu

# Write a complete Nginx site config using 'tee' and a here-document.
# We proxy HTTP traffic on port 80 to the Node app on localhost:3000 and pass through useful headers.
sudo tee /etc/nginx/sites-available/sparta-app >/dev/null <<'NGINX'
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:3000/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
NGINX

# Enable our site and disable the default one, then validate and reload Nginx
sudo ln -sf /etc/nginx/sites-available/sparta-app /etc/nginx/sites-enabled/sparta-app
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx

# Final note: the app may show DB errors until DB_HOST is set at launch (via User Data) or you point it to a running MongoDB.
echo "Build complete. Set DB_HOST at launch time so the app can talk to MongoDB."
```
**Run it:**

```bash
chmod +x app_script.sh
sudo source app_script.sh
```

**Quick checks:**

```bash
pm2 status
curl -I http://localhost
sudo nginx -t
```

- Note:
  
The app may log database errors at this stage because DB_HOST is not set. That is expected. You will set it via User Data on launch.

## 2) Create the App AMI

- Using the AWS console:

- Select the configured App instance

- Actions → Image and templates → Create image

- Name it <mark>node-app-ubuntu-22-04-v1</mark>

- Leave Reboot enabled and create the image

- Wait until the AMI status is Available


## 3) Launch from the App AMI with User Data

```bash

#!/bin/bash

# 1. Start with the shebang
# Tells EC2 to run the script with Bash

# 2. Move to the home directory
cd /home/ubuntu
# A safe working directory for scripts and files

# 3. Clone the app from GitHub
git clone https://github.com/LSF970/se-sparta-test-app.git
cd se-sparta-test-app/app
# Pulls the project code and moves into the app folder

# 4. Wait a bit to ensure previous steps completed
sleep 10

# 5. Install dependencies
npm install
# Installs Node.js packages listed in package.json

# 6. Wait for MongoDB to be ready
sleep 10
# Gives the database time to start up

# 7. Set the MongoDB environment variable
export DB_HOST=mongodb://localhost:27017/posts
# If MongoDB is on another EC2, use its private IP for example:
# export DB_HOST=mongodb://<PRIVATE_IP>:27017/posts

# 8. Seed the database
node seeds/seed.js
# Inserts sample data so /posts works immediately

# 9. Wait for seeding to finish
sleep 10

# 10. Start the app with PM2 and persist it
pm2 start app.js --name sparta-app
pm2 save
pm2 startup systemd -u ubuntu --hp /home/ubuntu
# Runs the app in the background and restores it on reboot

# 11. Restart Nginx to ensure the reverse proxy is active
sleep 10
sudo systemctl restart nginx
# Nginx proxies requests from port 80 to the app on port 3000
```

## 4) Test

- Open `http://<App-EC2-Public-DNS>/`

- Open `http://<App-EC2-Public-DNS>/posts to confirm seeded data`

## 5) Troubleshooting

- App not reachable on HTTP

  - Check security group allows inbound port 80

  - sudo nginx -t and sudo tail -n 200 /var/log/nginx/error.log

- App process not running

  - pm2 status and pm2 logs sparta-app

  - Re-run pm2 start app.js then pm2 save

- Database connection errors

  - Ensure DB_HOST uses the database private IP if DB is on a separate EC2

  - Confirm DB security group allows 27017 from the App instance or its security group
