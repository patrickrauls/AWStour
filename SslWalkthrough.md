# Patrick Rauls - AWS tour - now with SSL!

This is a walkthrough that demosntrates how to use continuous integration and deployment for your github project to AWS EC2. And when we say AWS EC2, we mean that we are not doing anything fancy with auto-provisioning/scaling, etc - just a simple bare-bones deployment.  

### Pre-requisites and considerations
You will need an account at AWS  
ES6 is not being used for this particular project (but it could be used)  
We are using Postgresql  
Must be using express.static in your server/app file  
We will be storing sensitive environment variables in a .env file  
* Ensure your .env file is included within your .gitignore file  
* **This may not be the most secure method of handling env vars - but that can be explored at another time**  

### Create an EC2 instance on AWS
```
Go to AWS EC2 dashboard
Select 'Launch Instance' 
Select Ubuntu Server 16.04
Select t2.micro -> Configure Instance Details
For now you can leave the defaults for configuration - you can always add roles, etc later
Click on "Next: Add Storage"
Leave at default for free tier -> Add Tags
It is suggested to add a name key to your instance - just makes it easier to find in the future - but not required
Hit "Next: Configure Security Group"  

Create a new security group
We will need to allow the following inbound traffic rules to the instance. You can use default for the source 
SSH - Generally, I prefer to use My IP as the source - select "My IP" from the dropdown
HTTP
HTTPS
PostgreSQL

Review and Launch
Launch!

You need to create a key/pair and download that to your machine. 
***You will not be given another opportunity to download this key/pair. 
Alternatively, you may use an existing key pair.

Launch Instance-> View Instances - it will take a few minutes for this to provision
```

### Assign proper permissions to your key/pair
At this point, your new key/pair is in your downloads folder. You probably do not want to work from this folder (although it is possible). I prefer to store all of my keys in the folder ~/.ssh. So, in my case, I would copy the key/pair to that dir and begin a terminal session in that folder.

* note: within your ssh session, you may experience a situation where the terminal no longer responds when you type. Chances are your session has timed out and you will eventually receive a "broken pipe" message. Just wait for the prompt to reappear and login again using the "ssh -i ..."  command listed below

From the terminal
```
# navigate to your working directory
cd ~/.ssh
# move the file to your working directory - note the dot at the end of the command to move to the current dir
mv ~/Downloads/myWalkthroughPair.pem .
# assign proper permissions
chmod 400 myWalkthroughPair.pem

# get the public DNS info from your AWS instance
# we will be logging in with the user ubuntu
# example ssh -i "[your key name].pem" [user]@[instance public DNS]
ssh -i "myWalkthroughPair.pem" ubuntu@ec2-xx-xx-xx-xx.compute-1.amazonaws.com

# you will need to confirm the addition of the new host 
yes

# You may promote your user to root by issuing sudo su - but it is not recommended. 
# We will be issuing commands with different users during this walkthrough

# first order of business is to update and upgrade the instance - apply any security patches, etc
sudo apt-get update && apt-get upgrade

#######################
# TODO Patrick?
I am receiving the following error upon running the last command:
Reading package lists... Done
E: Could not open lock file /var/lib/dpkg/lock - open (13: Permission denied)
E: Unable to lock the administration directory (/var/lib/dpkg/), are you root?
# Can I safely ignore?
#######################

# create a shell script that will define the node installation
curl https://deb.nodesource.com/setup_7.x -o node.sh

# confirm the shell script has been added correctly
ls
# if you want to see the files innards - nano node.sh
# run the script
sudo bash node.sh

# install node proper and npm
sudo apt-get install node.js
# confirm
y

# also install debian package dev tools
sudo apt-get install build-essential
# confirm
y

# PYTHON issues
# python 3 has been automatically installed, but if you are using bcrypt 
# (you are using bcrypt aren't you?) you will need to install python 2.7
sudo apt-get install python2.7

# configure npm to use python 2.7
npm config set python /usr/bin/python2.7

###################
# TODO Patrick
# verify python install and correct configuration
TODO how to do this???
```

### Clone your repo to your instance - github Clone with HTTPS link!
Grab the https link from your github repo   
And from your existing terminal sesssion...  

```
# ensure you are working from your root dir and clone your project
cd ~
git clone https://github.com/[User Repo]/[Repo Name].git

# cd into your project's dir and npm install
cd [Repo Name]
npm i
```

### Install postgres

```
# global installs
# TODO you might need to do the global install via sudo bash
sudo npm i -g knex pg

# setup postgres db and create a "postgres" user
cd ~
sudo apt-get install postgresql postgresql-contrib
# confirm 
y

# switch user
sudo -i -u postgres

# enter postgres
psql

# create your database - don't forget to end your commands with a semi-colon
# SAVE YOUR INFORMATION - WRITE IT DOWN SOMEWHERE
CREATE DATABASE [db name];
# connect to your newly created database
\c [dbname]
# create a user on that db - be sure to save the password!!! - semi-colon!
CREATE USER [username] WITH SUPERUSER PASSWORD '[your password in here]';

# exit out of pg
\q

# exit out of postgres user
exit
```

### Setup your .env file
TODO is it possible to use the simpler connection string in production?
```
# create your .env file add in your key pairs and save

# you will probably need to change how you access your db for the instance
# locally, we are able to use simple connection strings, but in the instance,
# you may need to specify more info

# I went from using this locally
# DATABASE_URL=postgres://localhost:5432/[db name]

# to using this for the instance
# DB_USER=xxxx
# DB_PASSWORD=xxx
# DB=xxx              - the db name
# DB_PORT=5432
# DB_HOST=localhost

cd [project dir]

# edit your .env file
nano .env
# enter your environment variables


# ensure your knexfile.js file mirrors your changes to your new connection parameters
# As an example...

require('dotenv').config({silent:true});

module.exports = {
  development: {
   client: 'pg',
   connection: process.env.DATABASE_URL || 'postgres://localhost:5432/willcall'
 },
 production: {
   client: 'pg',
   connection: {
     database: process.env.DB,
     user: process.env.DB_USER,
     password: process.env.DB_PASSWORD,
     port: process.env.DB_PORT,
     host: process.env.DB_HOST
   }
 }
};
```

### Setup SSL
```
Visit (Certbot.org)[https://certbot.eff.org/]  
Enter "none of the above" for system  
Enter your system name as the ubuntu version you used for your instance - Ubuntu 16.04   
Use the listed install commands but DO NOT do the "GET STARTED" section yet - we will go there in a few steps
```
```
# from the root dir
cd ~

sudo add-apt-repository ppa:certbot/certbot
# confirm
[Enter]

# ensure everything up to date
sudo apt-get update

sudo apt-get install certbot 
# confirm
y
```

### Forever
Forever gives us the ability to run commmands and give us back the prompt  
AKA it allows us to run processes in the background  
```
# from the root dir
cd ~

# trying from [project dir]


sudo npm i -g forever

# add the following scripts to your package.json file
# next time you might set this up before deploying to github 
/*
"start": "node server.js",
"forever": "forever start server.js",
"again": "forever restart server.js",
"stop": "forever stopall",
"logs": "forever logs"
*/
cd [project dir]
nano package.json


# Remove? not ready for this yet
# run your server
# npm run forever
```

### TODO
make sure that your web site is being served at port 3000
make the move to 8000 after certs are installed

### IP Tables
Route incoming requests to the proper ports  
```
cd ~
sudo bash 
iptables -A PREROUTING -t nat -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 3000
iptables -A PREROUTING -t nat -i eth0 -p tcp --dport 443 -j REDIRECT --to-port 8000

# now to ensure that our iptables are persisted upon reboots, etc
sudo apt-get install iptables-persistent
# agree to prompts

# exit from root user
exit

# not ready to run server
```

### Setup DNS
Provision an Elastic IP for your instance on AWS  
```
From EC2 dashoboard>Elastic Ips>Allocate  
Select new Elastic IP>Actions>Associate address  
Select your instance - no need to specify private IP for this walkthrough  

Update your DNS entries at your Provider  
Use the public IP address from your instance

** keep in mind that it may take time for you to see your DNS changes propaget to the internet
You can use nslookup to see if your DNS is using the new values

nslookup [domainname]
# following is the response we are looking for
Server:		192.168.1.1
Address:	192.168.1.1#53

Non-authoritative answer:
Name:	[my domain name]
Address: xx.xx.xx.xx
# does this address match your new ip?
```

### Connect via new IP and complete CertBot
Your instance's ip will change after being associated with a new Elastic IP  
From your instance listing (EC2 Dashboard) select your instance and click the connect button  
SSH into your instance with the new connection information (should be using new IP)  
```
ssh -i myWalkthroughPair.pem ubuntu@ec2-x-x-x-x.compute-1.amazonaws.com

# confirm new host
yes 

# note that the IP at your prompt will reflect your instance's private IP address


# navigate to your project dir
cd [project]

# make sure your website is running on port 3000 - whatever you have specified in the iptables
# for port 80 traffic. We will setup serving requests over ssl in a later step
# verify by running the following!
node server.js
# you should see this
Express server listening on port 3000

# now stop the server
ctrl-c

# run your site with forever to setup the certificates
forever start server.js

# you should be able to visit your site in a browser using your domain name
# you will not be able to continue until this is setup properly

#####################
# setup certbot
#####################
sudo certbot certonly

# options 
# 1 place files in a webroot directory
# enter email address
# Agree
# up to you to share your email

# domain name - Be sure to include @ and www domain names!
mydomain.com, www.mydomain.com

# 1 enter a new webroot 
1
# Input the webroot
public

# select corresponding number from the list
2
Waiting for verification...

# If you fail too many attempts - 5 per hour? you will need to wait an hour to try again
# for more details, visit https://community.letsencrypt.org/t/rate-limiting-due-to-authz-errors/31632

# letsencrypt logfiles are located at /var/log/letsencrypt and you will need to access as root



# All Set!!! 

# Now configure keys - must access keys as root
cd ~
sudo bash
mkdir keys

########################
# TODO Patrick
# I am pretty sure I just need to do this for my naked domain
# can you verify that I do not need to do this for the www.[domain name]?
########################

# certbot will have placed your keys into the direcories listed below 
# copy them up to the keys diyour server's root
cp /etc/letsencrypt/live/[your domain name]/fullchain.pem keys/
cp /etc/letsencrypt/live/[your domain name]/privkey.pem keys/

exit
```


### Setup server serving assets to port 8000
Now that they keys are residing in your server's root/keys dir, edit your server.js file to serve correctly  
In my case, I will need to add a few things to my server.js. Ultimately, I should just add some conditionals to my server.js to handle the case of ```if NODE_ENV=production```  
You may also need to adjust your .env file to serve from the correct port  
```
cd ~
cd [project dir]
nano server.js

// add imports/requires
var fs                  = require('fs');
var http                = require('http');
var https               = require('https');


// prior to setting up the view engine and declaring the static path
// declare a server, and redirect all requests to non-https to https
// assign cert keys to server
http.createServer(app).listen(3000);

app.all('*', function (req,res,next) {
	return req.secure ? next() : res.redirect('https://' + req.hostname +  req.url)
});

var server = https.createServer({
	cert: fs.readFileSync('../keys/fullchain.pem'),
	key: fs.readFileSync('../keys/privkey.pem')
}, app);



// towards the bottom of your server.js,  change this line
app.listen(app.get('port'), function() {
  console.log('Express server listening on port ' + app.get('port'));
});

module.exports = app;

// to the following
server.listen(app.get('port'), function() {
  console.log('Express server listening on port ' + app.get('port'));
});

module.exports = app;
```


### Getting the darn thing to run already
```
# get into your project dir if you aren't already
cd [project dir]

# migrate the database
# be sure to specify the environment! or else you will get the error below!
knex migrate:latest --env=production

# if you receive the following error...you probably did not specify the environment properly
Knex:warning - Pool2 - Error: Pool was destroyed
Knex:Error Pool2 - error: password authentication failed for user "ubuntu"
....

# seed the database - note environment stipulation from above
knex seed:run --env=production


# run forever scripts - examine logs - assumes you have setup forever scripts in your package.json
# stop the webserver or restart
npm run again

# list the log files
npm run logs
# this will list the filename where the logfiles reside
# one method to view
more /home/ubuntu/.forever/[logfile name].log
```

