# Ssl Walkthrough Checklist

### Key
```
wd      = working directory
aws     = AWS Console
~       = instance user dir
~/pd    = project directory
```

### Create EC2 instance
- [] aws Security Group - allow SSH, HTTP, HTTPS, PostgreSQL
- [] Save new key pair or use existing
- [] wd mv ~/Downloads/key-pair.pem . [note dot at end!]
- [] wd chmod 400 key-pair.pem

### SSH into your instance and configure environment
- [] Gather connect details by highlighting instance and click on connect button
- [] wd ssh -i "key-pair.pem" ubuntu@ec2-[your-ip-address].compute-1.amazonaws.com
- [] ~ sudo apt-get update && sudo apt-get upgrade
- [] ~ curl https://deb.nodesource.com/setup_7.x -o node.sh
- [] ~ sudo bash node.sh
- [] ~ sudo apt-get install node.js
- [] ~ sudo apt-get install build-essential
- [] ~ sudo apt-get install python2.7
- [] ~ npm config set python /usr/bin/python2.7

### Install PostgreSQL
- [] ~/pd sudo npm i -g knex pg
- [] ~ sudo apt-get install postgresql postgresql-contrib
- [] ~ sudo -i -u postgres
- [] ~ psql
- [] psql> CREATE DATABASE [db name];
- [] psql> \c [dbname]
- [] *SAVE myPassword somewhere memorable!*
- [] psql> CREATE USER [username] WITH SUPERUSER PASSWORD '[myPassword]';
- [] \q

### Clone your repo to instance - *Use HTTPS link*
- [] ~ git clone https://github.com/[User Repo]/[Repo Name].git
- [] ~/pd npm install

### Configure .env and knexfile.js
- *~/pd nano .env && ~/pd knexfile.js*
```
.env DATABASE_URL=postgres://[user]:[password]@[host]:[port]/[dbname]
.knexfile.js -> connection: process.env.DATABASE_URL

*-OR-*

.env -> DB, DB_USER, DB_PASSWORD, DB_PORT, DB_HOST
knexfile.js -> connection: {
    database: process.env.DB,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    port: process.env.DB_PORT,
    host: process.env.DB_HOST
}
```

### SSL setup
- [] https://certbot.eff.org/ -> software - none && system -> instance's OS -> ubuntu 16.04
- [] ~ sudo add-apt-repository ppa:certbot/certbot
- [] ~ sudo apt-get update
- [] ~ sudo apt-get install certbot

### Forever
- [] ~/pd sudo npm i -g forever
- [] ~/pd nano package.json
```
"start": "node server.js",
"forever": "forever start server.js",
"again": "forever restart server.js",
"stop": "forever stopall",
"logs": "forever logs"
```

### Make sure your server will run on your port before continuing
- [] ~/pd node server.js

### IP Tables
- [] ~ sudo bash
- [] ~ iptables -A PREROUTING -t nat -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 3000
- [] ~ iptables -A PREROUTING -t nat -i eth0 -p tcp --dport 443 -j REDIRECT --to-port 8000
- [] ~ sudo apt-get install iptables-persistent
- *Exit from root user*
- [] ~ exit

### DNS
- [] aws dashboard>Elastic Ips>Allocate
- [] aws select new Elastic IP>Actions>Associate address>Select your instance

### Connect via new IP and complete Certbot
- [] wd ssh -i "key-pair.pem" ubuntu@ec2-[your-elastic-ip-address].compute-1.amazonaws.com
- [] *Verify port*
    - [] ~/pd node server.js
    - [] ~/pd [ctrl-c]
- [] ~/pd npm run forever
- [] ~/pd sudo certbot certonly
    - domain list - include naked and www.
    - webroot - public
- [] ~ sudo bash
- [] ~ mkdir keys
- [] ~ cp /etc/letsencrypt/live/[naked domain]/fullchain.pem keys/
- [] ~ cp /etc/letsencrypt/live/[naked domain]/privkey.pem keys/
- [] ~ exit

### Configure Server.js to use port 8000
- [] ~/pd nano server.js
```
var fs                  = require('fs');
var http                = require('http');
var https               = require('https');

...
// prior to setting up the view engine and declaring the static path
http.createServer(app).listen(3000);

app.all('*', function (req,res,next) {
	return req.secure ? next() : res.redirect('https://' + req.hostname +  req.url)
});

var server = https.createServer({
	cert: fs.readFileSync('../keys/fullchain.pem'),
	key: fs.readFileSync('../keys/privkey.pem')
}, app);

...
// change app.listen to ->
server.listen(app.get('port'), function() {
  console.log('Express server listening on port ' + app.get('port'));
});
```

### Set the port in .env to 8000 (if applicable)
- [] ~/pd nano .env

### Final steps
- [] ~/pd knex migrate:latest --env=production
- [] ~/pd knex seed:run --env=production
- *Restart server*
- [] npm run again
- [] npm run logs
- [] more /home/ubuntu/.forever/[logfile name].log