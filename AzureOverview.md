# Overview of Azure

## What's Azure's market share?


# Structure

Resources need to be contained within a resource group.

Resource groups are associated with subscriptions, that limit you on what you can create within resource groups depending on how much you pay.

Management groups can set policies that affect everything below it.
- However, policies can be set at each level too, that filter down to levels below them
- Management groups have layers/levels too

Each of these layers is called a **scope**.

![Alt text](image-1.png)

# AWS and Azure differences

- Naming
  - VPC is called virtual network, or vnet in Azure.
  - CIDR blocks are called IPv4 address space
  - Instances are called Virtual machines
  - AMIs are called images

- Availability Zones
  - Azure has maximum of 3 AZs per region, where AWS can have more depending on the region
  - In Azure you can select multiple AZs for an instance, which will make a copy per zone selected

# Virtual Networks

## Making the virtual network and subnets

To create one...
1. Search vnet and select virtual network
2. Create virtual network and under basics make sure resource group is tech254 and region is UK South

![Alt text](image.png)

3. Go to IP adresses tab and make subnets. Make public and private, and choose appropriate CIDR blocks and name. Leave security groups for now.

![Alt text](image-2.png)

4. Then we can tag it with our name as owner, for others to know who made this.

![Alt text](image-3.png)

## Making the virtual machines

### Client tier

**NOTE:**: In an actual 2-tier virtual network, we will set up the database first. I only go over the client tier first as an example.

5. To make an virtual machine on our virtual network, we go to virtual machines and click create dropdown and the first option.

![Alt text](image-4.png)

6. We then select the correct resource group, a good name and AZ 1. 

![Alt text](image-5.png)

7. We want ubuntu pro 18.04 LTS generation 2 and security group standard. Double check this is selected, there's a bug that causes this to revert to 20.04.

![Alt text](image-6.png)

![Alt text](image-7.png)

8. Under size, select see more sizes and B1S.

![Alt text](image-8.png)

9. Under administrator account, change the username and under SSH public key source, click use selected key used in azure and select my key.

![Alt text](image-9.png)

10. Select HTTP and SSH for selected ports (dont need 3000 as rev proxy works). This is for our app instance so we want these open.

![Alt text](image-10.png)

11. Go to disks tab and change OS disk type to standard SSD.

![Alt text](image-11.png)

12. Go to networking tab and pick our virtual network, give it a name and tick delete public IP and NIC when VM is deleted.

![Alt text](image-12.png)

13. Go to advanced, and select user data then paste it. We are using the following user data like we did in AWS. 

```
#!/bin/bash

# update
sudo apt-get update -y

# upgrade, this requires this command to do it without user input in Azure
sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y

# install nginx and sed
sudo apt install nginx -y
sudo apt install sed -y

# clone app from repo (make specific repo for app later)
cd /home/adminuser
sudo -u adminuser git clone https://github.com/jbjoeburns/CI_CD.git

# node js 12.x installed
curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
sudo apt-get install -y nodejs

# install node 
sudo npm install

# rev prox
sudo sed -i "s/try_files \$uri \$uri\/ =404;/proxy_pass http:\/\/localhost:3000\/;/" /etc/nginx/sites-available/default

# env variable
export DB_HOST=mongodb://<public IP for db>:27017/posts

# other packages we need
sudo npm install express
sudo npm install mongoose
sudo npm install ejs
sudo npm install faker

# move to app directory + restart nginx
cd /home/adminuser/CI_CD/app/app
sudo systemctl restart nginx

# install pm2 
sudo npm install pm2 -g
npm install

# kill, in case anything else is running
sudo pm2 kill

# start app
sudo pm2 start app.js

# seed db
sudo node seeds/seed.js
```

![Alt text](image-13.png)

- **NOTE:** It's worth mentioning that `DEBIAN_FRONTEND=noninteractive` needs to be put infront of `apt upgrade -y`. The reason for this is that a certain package installed by default in Azure requires additional configuration on install, that asks for manual user input and interrupts our upgrade progress. As we aren't using this package, we can put this option in to skip over the configuration steps avoiding the user input for increased automation.

14. Go to tags tab and put in owner then our name.

![Alt text](image-14.png)

Then we're done! 

**HOWEVER** the user data takes 5-10 mins to run in it's entirety. Give it time to install everything, and if you want to check where in the script it's up to, you can use `<package> --version` to see what has been installed so far.

## Database tier

The set up is almost identical to that of the client tier, with the following changes.

1. In basics, choose a different availability zone, so downtime in the client tier's AZ won't affect the database tier, and vice versa.

![Alt text](image-15.png)

2. In networking, make sure public IP is set to none. We do not want a public IP for security reasons, making it so hackers cannot access our database easily. Additionally, we do not need one as the client tier can connect to the database tier through our VN.

![Alt text](image-16.png)

3. Still in networking, we want to select **advanced** under NIC network security group, then create new network security group. Then we want to add a new inbound rule with the destination port of 27017, the mongoDB port.

![Alt text](image-17.png)

![Alt text](image-18.png)

4. Then in advanced, we use the following user data...

```
#!/bin/bash

# update/upgrade
sudo apt update
sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y

# acquire key to mongodb version we want (3.2).
wget -qO - https://www.mongodb.org/static/pgp/server-3.2.asc | sudo apt-key add -

# verify the key.
echo "deb http://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.2.list

# update our app list again
sudo apt update

# install mongodb
sudo apt-get install -y mongodb-org=3.2.20 mongodb-org-server=3.2.20 mongodb-org-shell=3.2.20 mongodb-org-mongos=3.2.20 mongodb-org-tools=3.2.20
sudo apt-get install -y mongodb-org=3.2.20 mongodb-org-server=3.2.20 mongodb-org-shell=3.2.20 mongodb-org-mongos=3.2.20 mongodb-org-tools=3.2.20

# edit the config to define what IP we want to be able to connect to the database.
sudo sed -i 's/bindIp: 127.0.0.1/bindIp: 0.0.0.0/' /etc/mongod.conf

# start mongoDB.
sudo systemctl start mongod
sudo systemctl enable mongod

# check its working
sudo systemctl status mongod
```