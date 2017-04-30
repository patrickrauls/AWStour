# Setting up EC2 server - both front and back
https://console.aws.amazon.com/ec2/

1. Launch an instance
2. Select Ubuntu Server 16.04 LTS (HVM), SSD Volume Type - ami-efd0428f
  - Installs Python 3, _which may be incompatible with dependencies_
  - (free tier as of 2017)
3. Select the t2.micro type, if not already selected by default.
_(assumes that this instance type is still eligible for the free tier.)_
  - __Next__ Configure Instance Details
4. IAM roles can be added later
  - __Next__ Add Storage
5. Storage - accept default for now (free tier eligible customers can get up to 30GB)
  - __Next__ Add Tags
6. Tags - Accept default for now
  - __Next__ Configure Security Group
7. Configure Security Group (aka Firewall rules)
    - Assign a security group:
      1. Select Create New or Use Existing, accordingly
      2. Rename "launch-wizard1" to something useful like "ssh-http"
      3. Add Rule
        - Select HTTP     
        - HTTPS security not worth the headache at this time
          - Costs Money
          - This will be covered later in video series
  - __Review and Launch__
8. Review Launch
  - __Launch__
9. Create new keypair
  - Amazon EC2 uses public–key cryptography to encrypt and decrypt login information; creates a key in a file.pem format (RFC 1421-RFC 1424)
    - Recommend choosing a meaningful name for file.pem, possibly after the project _(e.g. shook.pem)_
    - Not recoverable, so store safely on your local machine:
      1. __cd__ location of file.pem
      2. chmod 400 file.pem
        - _Changes the permissions of the .pem file so only the root user can read it._
      3. For additional security, move file to secure (chmod <600) and/or hidden folder
10. Link information for securely connecting to remote host
    1. Copy the entire link _(starting with "ssh i")_
      - This link can also be found by:
        1. Go to EC2 instance
        2. Click on the Connect button
    2. Paste into console window __Enter__
    3. Type __yes__ to allow access from IP
    (http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html)


# On remote server
1. sudo apt-get update && sudo apt-get upgrade
  - Type __yes__ to confirm
2. Install package maintainers
3. curl https://deb.noderesource.com/setup_7.x -o nodesource_setup.sh

  _Curl is a tool for transferring data from or to a server,  and is designed to work without user interaction.
  In this case, we are grabbing the setup script for node7 and plugging it into this shell script_
4. sudo bash nodesource_setup.sh
  _Starts a bash shell as root level user to run the script_
5. sudo apt-get install nodejs
  _apt-get utility is a powerful and free package management command line program, that is used to work with Ubuntu’s APT (Advanced Packaging Tool) library to perform installation of new software packages, removing existing software packages, upgrading of existing software packages and even used to upgrading the entire operating system._
  - __node -v__ to confirm that we are using version 9
6. sudo apt-get install build-essential
  - Packages needed to compile a debian package:
    - Generally includes the gcc/g++ compilers an libraries and some other utils.
    - Also necessary if need to build or compile using C/C++
  - Type __yes__  to set up necessary space
  - __npm -v__ to confirm we are using node version 4.2
7. sudo bash iptables -A PREROUTING -t nat -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 3000
  _sets up a transparent proxy (firewall/NAT rules) to redirect traffic coming in on port 80 to port 3000 instead_ https://www.cyberciti.biz/faq/linux-port-redirection-with-iptables/
8. apt-get install iptables-persistent
  _allows us to save our rule sets and have them automatically applied at boot_
  - Type __yes__ to save current rules
9. CTRL-D to get out of sudo / root
Server is essentially ready
  _git clone your repo_

# Database
1. sudo apt-get install postgress postrgresql-contrib
  - _to establish database connection_
  - Type __yes__ to allocate necessary space
2. sudo -i -u postgres

3. psql
4. create user ubuntu with superuser;
  - Using (name) Ubuntu allows direct access to database using __psql__ at command line rather than __sudo -i -u postgres__
5. Alter role ubuntu with password '^catwalk^';
  - __^catwalk^__ is typing random string, as though a cat walked across your keyboard
  - Type __\q__ to get out of alter
  - __exit__ to get out of psql
6. sudo npm install -g pg
  - Type __psql__ to confirm that psql drops right into postgres interface for ubuntu (user) database
  - Add the database to gitignore
  - Add the database username/password to ENV file



# Elastic IP
  _Masks the failure of an instance or software by rapidly remapping the address to another instance in your account._

1. Go to AWS, then EC2 dashboard
2. Click link for Elastic IP
3. Click button for allocate new address (left)
  - Click __Allocate__  button (right)
    - You will receive a new Elastic (public) IP, which will also cause the associated ssh remote connection link to change

4. click actions (dropdown)
  - select Associate Address
      - Resource Type:   Instance
      - Instance:        (select what should be the only ID entry)
      - Private IP:      (select what should be the only IP entry)
  - Click __Associate__ (button)
      -  A new Private IP will be associated, with your instance


# Update DNS
#### Route53 (part 2)
1. Create Record Set (button)
2. Create Record Set (right side)
  1. Wildcard  (record that matches requests for non-existent subdomains: )
    - Name:      *
    - Type:      A
    - Alias:    no
    - Value:    public IP (e.g. 54.148.158.100)
    - __Create__
  2. Default    (allows site resolution if user fails to add www or other subdomain)
    - Name:      *
    - Type:      A
    - Alias:    no
    - Value:    public IP (e.g. 54.148.158.100)
    - __Create__


# Persistent (node) Web Site
#### Install Forever
1. Securely connect to remote EC2 instance
2. sudo npm install -g forever
  - _keeps server alive_
3. npm start

#### Update package.json in git
Edit __scripts__ to include:
  - "forever": "forever start app.js"
  - "again": "forever restart app.js"
  - "stop": "forever stopall"

#### CLI usage:
  - npm run forever
    - _to keep server running_
  - npm run again
    - _to restart server_
  - npm run stop
    - _to stop server_


# Misc Notes

#### Lambda
Allows you to trigger a function
  - Router.POST
  - Basically gives you a repl

#### iptables
- To stop
  - $ sudo service ufw stop
- To start
  - $ sudo service ufw start
