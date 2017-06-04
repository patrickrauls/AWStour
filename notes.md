# Setting up EC2 server - both front and back

1. Login or create [AWS](https://console.aws.amazon.com/ec2/) account to create EC2 instance.
2. Launch Instance (button)
3. Select Ubuntu Server 16.04 LTS (HVM), SSD Volume Type - ami-efd0428f
    - Installs Python 3, _which may be incompatible with dependencies_
     - (free tier as of 2017)
4. Select the t2.micro type, if not already selected by default.
_(assumes that this instance type is still eligible for the free tier.)_
    - __Next__ Configure Instance Details
5. IAM roles can be added later
     - __Next__ Add Storage
6. Storage - accept default for now (free tier eligible customers can get up to 30GB)
     - __Next__ Add Tags
7. Tags - Accept default for now
     - __Next__ Configure Security Group
8. Configure Security Group (aka Firewall rules)
    - Assign a security group:
      1. Select Create New or Use Existing, accordingly
      2. Rename "launch-wizard1" to something useful like "ssh-http"
      3. Add Rule
            - Select HTTP     
            - HTTPS security not worth the headache at this time
              - Costs Money
              - This will be covered later in video series
    - __Review and Launch__
9. Review Launch
    - __Launch__
10. Create new keypair
    - Amazon EC2 uses public–key cryptography to encrypt and decrypt login information; creates a key in a file.pem format (RFC 1421-RFC 1424)
     - Recommend choosing a meaningful name for file.pem, possibly after the project _(e.g. shook.pem)_
    - Not recoverable, so store safely on your local machine:
      1. `__cd__` location of file.pem
      2. `chmod 400` file.pem
            - _Changes the permissions of the .pem file so only the root user can read it._
      3. For additional security, move file to secure (chmod <600) and/or hidden folder
11. Link information for securely connecting to remote host
    1. Copy the entire link (starting with "ssh i"). &nbsp; _This link can also be found by clicking on the _Connect_ button in the EC2 Instance._
    2. Paste into console window, then __Enter__
    3. Type __yes__ to allow access from IP
    _See:_ [Connecting to Your Linux Instance Using SSH](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html)


# On remote server
1. Update Ubuntu
    ```
    sudo apt-get update && sudo apt-get upgrade
    ```
    - Type __yes__ to confirm (use of space)
2. Package Configuration
     - Select : _Install package maintainer's version_
3. Grab the setup script for node7 and plug it into this shell script
    ```
    curl https://deb.nodesource.com/setup_7.x -o nodesource_setup.sh
    ```
    _Curl is a tool for transferring data from or to a server,  and is designed to work without user interaction._
4. Starts a bash shell as root level user to run the script
    ```
    sudo bash nodesource_setup.sh
    ```
5. Install node.js
    ```
    sudo apt-get install nodejs
    ```

    _apt-get utility is a powerful and free package management command line program, that is used to work with Ubuntu’s APT (Advanced Packaging Tool) library to perform installation of new software packages, removing existing software packages, upgrading of existing software packages and even used to upgrading the entire operating system._
     - Confirm what version of node being used: (e.g. 7.8.0)
      ```
      node -v
      ```

6. Install build essentials
    ```
    sudo apt-get install build-essential
    ```
      - Packages needed to compile a debian package:
        - Generally includes the gcc/g++ compilers an libraries and some other utils.
        - Also necessary if need to build or compile using C/C++
      - Type __yes__  to set up necessary space
      - Confirm which version of npm being used: (e.g 4.2)
      ```
      npm -v
      ```
7. Setup bash environment
      ```
      sudo bash
      ```
8. Set up a transparent proxy (firewall/NAT rules)
      ```
      iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 3000
      ```
     - Redirects traffic coming in on port 80 to port 3000 instead: see  [Linux Port Redirection](https://www.cyberciti.biz/faq/linux-port-redirection-with-iptables/)
9. Set proxy rule(s) to be automatically applied at boot
      ```
      apt-get install iptables-persistent
      ```
      - Select __yes__, and/or type __yes__ to save current rules for IPv4.
      - Select __yes__, and/or type __yes__ to save current rules for IPv6.
10. CTRL-D to get out of sudo / root
      - Server is essentially ready
11.  Git clone your repo

# Database
1. Establish database connection
    ```
    sudo apt-get install postgresql postgresql-contrib
    ```
    - Type __yes__ to allocate necessary space
2. Install postgres
    ```
    sudo -i -u postgres
    ```
3. Drop into postgres
    ```
    psql
    ```
4. Create Ubuntu user
    ```
    create user ubuntu with superuser;
    ```
      - Using (name) Ubuntu allows direct access to database using __psql__ at command line rather than `sudo -i -u postgres`
5. Create password for role
    ```
    Alter role ubuntu with password `^catwalk^`;
    ```
      - __`^catwalk^`__ is typing random string, as though a cat walked across your keyboard
      - Type `\q` to get out of alter
      - Then `exit` to get out of psql
6. Install postgress
    ```
    sudo npm install -g pg
    ```
7. Check database
      - Type `psql` to confirm that psql drops right into postgres interface for ubuntu (user) database
      - if does not exits, then `createdb ubuntu`
8. Tying db to app
      - Add the database to .gitignore
      - Add the database username/password to ENV file



# Elastic IP
  _Masks the failure of an instance or software by rapidly remapping the address to another instance in your account._

1. Login to EC2 dashboard
2. Click _Elastic IP_ (link)
3. Click _Allocate New Address_ (button on left)
      - Click _Allocate_  (button right)
        - You will receive a new Elastic (public) IP, which will also cause the associated ssh remote connection link to change
    - Click __Close__
4. Click _New IP_ (link)
5. Click _Actions_ (dropdown)
      - Select Associate Address
          - __Resource Type:__   Instance
          - __Instance:__       (select the desired ID entry)
          - __Private IP:__     (select the desired IP entry)
      - Click _Associate_ (button)
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
    - Click __Create__ (button)
  2. Default    (allows site resolution if user fails to add www or other subdomain)
    - Name:      *
    - Type:      A
    - Alias:    no
    - Value:    public IP (e.g. 54.148.158.100)
    - Click __Create__ (button)


# Persistent (node) Web Site
#### Install Forever
1. Securely connect (ssh) to remote EC2 instance
2. Install "forever" module
    ```
    sudo npm install -g forever
    ```
3. Start the server
    ```
    npm start
    ```

#### Update package.json in git
Edit package,json __scripts__ to include forever, again, and stop modules:
```
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "start": "node ./bin/www",
    "forever": "forever start ./bin/www",
    "again": "forever restart ./bin/www",
    "stop": "forever stopall"
```

#### CLI usage:
1. To keep the server running
      ```
      npm run forever
      ```
2. To restart the server
      ```
      npm run again
      ```
3. To stop the server
      ```
      npm run stop
      ```


# Misc Notes

#### Lambda
1. Allows you to trigger a function (e.g.router.POST)
2. Basically gives you a repl


#### iptables
1. To stop using rules
    ```
    sudo service ufw stop
    ```
2. To start/restart using rules
    ```
    sudo service ufw start
    ```
