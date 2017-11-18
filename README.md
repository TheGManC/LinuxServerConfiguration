# Linux Configuration

## Server
- IP address: 52.31.4.176

- Accessible SSH port: 2200

- Application URL: http://ec2-52-31-4-176.eu-west-1.compute.amazonaws.com

- Follow the instructions provided to SSH into your server.
`ssh -i ~/.ssh/lightrail_key.rsa ubuntu@52.31.4.176`

## Secure Your Server
1. Update all currently installed packages.
 - update package source list `sudo apt-get update`
 - update software - `sudo apt-get upgrade`
 - Optional :remove packages no longer required - `sudo apt-get autoremove`
 - Optional: install finger `sudo apt-get install finger`. Finger shows details of all currently logged in users.

2. Change the SSH port from 22 to 2200. Make sure to configure the Lightsail firewall to allow it.
 - `sudo nano /etc/ssh/sshd_config` and change `Port` to `2200` instead of `22`. Quit and Save.

3 . Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

Ubuntu comes with a firewall preinstalled called ufw but it is deactive by default. `sudo ufw status` will verify this.

 - Start off by denying everything: `sudo ufw default deny incoming`
 - Allow all outgoing by default: `sudo ufw default allow outgoing`
 - `sudo ufw allow ssh` allows ssh connections
 - Allow all TCP connections for port 2200 `sudo ufw allow 2200/tcp`
 - Allow basic html connections `sudo ufw allow www`
 - Allow NTP `sudo ufw allow 123/udp`
 - deny port 22 as our default is now 2200 `sudo ufw deny 22`
 - Enable our fire wall `sudo ufw enable`
 - Our firewall is now actice, this can be verfified by running `sudo ufw status`

*Warning: When changing the SSH port, make sure that the firewall is open for port 2200 first, so that you don't lock yourself out of the server. Review this video for details! When you change the SSH port, the Lightsail instance will no longer be accessible through the web app 'Connect using SSH' button. The button assumes the default port is being used. There are instructions on the same page for connecting from your terminal to the instance. Connect using those instructions and then follow the rest of the steps.*

 - Now update Amazon Lightsail instance firewall to:
   - TCP -> 80
   - UDP -> 123
   - TCP -> 2200

4 . Give grader access. **In order for your project to be reviewed, the grader needs to be able to log in to your server.**

- Create a new user account named grader.
 `sudo adduser grader`
- Optional: Confirm that the user has been created by running the `finger` command : `finger grader`
- switch to grader `su - grader` and when prompted, insert the graders password which was set when creating the user.

 Currently grader does not have persmission to do the sudo command. Running `sudo cat /etc/passwd` (as grader) and inputing your password will output the following: `grader is not in the sudoers file.  This incident will be reported.`

5 . Give grader the permission to sudo.
 
Open a seperate terminal window and log in as ubuntu.
Create a file `sudo nano /etc/sudoers.d/grader` and paste in the following: `grader ALL=(ALL:ALL) ALL` and save and quit.
Go back to the terminal that has grader logged in, running
`sudo cat /etc/passwd` will now show the list.

6 .  Create an SSH key pair for grader using the `ssh-keygen` tool. Passwords arent very secure so we should use another authentication method.

- Generate a key pair on your local machine, never on the server as you cant be sure that the private key is indeed private.
 - `ssh-keygen` generates the key pair
 - you will be asked to give a file name for the key pair e.g `/Users/User/.ssh/linuxCourse`
 - you will be then asked for a passphrase
 - You will then notice that too files have been generated `/Users/User/.ssh/linuxCourse` and a public key `/Users/User/.ssh/linuxCourse.pub`. This public key is what we will place on our server to enable key based authentication.

  **Installing a public key**

  - we need to put our local public key onto the server
 - Make sure you are the `grader`
 - `mkdir .ssh` (this is the directory where all your keys should be stored)
 - Create a new file `touch .ssh/authorized_keys` this is a special file where all of your public keys this account is allowed for authentication.
 - Now switch back to you local machine in terminal and print out the contents of the public key: `cat .ssh/linuxCourse.pub` and copy the output of the public key.
 - Now go back to grader. Open the authorized_keys file and paste in the output of the public key `nano .ssh/authorized_keys` and save and quit.
 - run `chmod 700 .ssh` and `chmod 644 .ssh/authorized_keys` to ensure other users cant gain access to your account.

 - We are now able to log in as the grader user but instead of using user name and password we can use the i flag and the key we want to use, will allow us to log in using this key pair: `ssh grader@52.31.4.176 -p 22 -i ~/.ssh/linuxCourse`. If you set a password for you key pair you will be asked for it.

 **Key-based SSH authentication is enforced.**
 - This will force all of your users to only be able to login using a key pair.
 - `sudo nano /etc/ssh/sshd_config` and find the line that reads `PasswordAuthentication` and make sure its value reads `no`, if it does not then change it. Quit and Save.
 - `sudo service ssh restart` to make apply changes

## Prepare to deploy your project.
Configure the local timezone to UTC.

- `sudo dpkg-reconfigure tzdata` and select `None of the above` from the presented menu, then select `UTC`
- Run `date` to confirm this.

### Install and configure Apache to serve a Python mod_wsgi application.
- Install apache `sudo apt-get install apache2`
- You can confirm this is working by visiting: `52.31.4.176` in your browser which should show the Ubuntu default page.

### Installing mod_wsgi
- Instal mod_wsgi `sudo apt-get install libapache2-mod-wsgi`
- You then need to configure Apache to handle requests using the WSGI module. You’ll do this by editing the `/etc/apache2/sites-enabled/000-default.conf` file. This file tells Apache how to respond to requests, where to find the files for a particular site and much more.
- add the following line at the end of the `<VirtualHost *:80>` block, right before the closing `</VirtualHost>` line: `WSGIScriptAlias / /var/www/html/myapp.wsgi`
- now restart apache `sudo service apache2 restart`

### Your First WSGI Application
- To quickly test if you have your Apache configuration correct you’ll write a very basic WSGI application. You just defined the name of the file you need to write within your Apache configuration by using the `WSGIScriptAlias` directive. Despite having the extension `.wsgi`, these are just Python applications. Create the `/var/www/html/myapp.wsgi` file using the command `sudo nano /var/www/html/myapp.wsgi`. Within this file, write the following application:

  `def application(environ, start_response):
    status = '200 OK'
    output = 'Hello Batman!'
    response_headers = [('Content-type', 'text/plain'), 
    ('Content-Length', str(len(output)))]
    start_response(status, response_headers)
   return [output]`

- This application will simply print return Hello Batman! in your browser.

- If you built your project with Python 3, you will need to install the Python 3 mod_wsgi package on your server: `sudo apt-get install libapache2-mod-wsgi-py3`


### Deploying a flask app on Ubuntu
- Install and enable mod_wsgi: `sudo apt-get install libapache2-mod-wsgi python-dev` and enable mod_wsgi with the following command: `sudo a2enmod wsgi`
- Create a Flask App in the /var/www directory by running `cd /var/www` and `sudo mkdir FlaskApp`. Move inside this folder `cd FlaskApp`.
- Once inside, create another directory name `FlaskApp`: `sudo mkdir FlaskApp`.
- Now move inside this directory and create two subdirectories:
`cd FlaskApp`, and to create the subdirectories `sudo mkdir static templates`
- Now create the __init__.py file that will contain the flask app logic:
`sudo nano __init__.py` and paste in the following:
`from flask import Flask
app = Flask(__name__)
@app.route("/")
def hello():
    return "Hello, I love Digital Ocean!"
if __name__ == "__main__":
    app.run()`
The save and quit.
- Install flask, and create a virtual environment for our flask application.
- Instal pip `sudo apt-get install python-pip`
- Instal the virtual env `sudo pip install virtualenv`
- Create a virtual env called `venv` by using the command `sudo virtualenv venv`
- Now install Flask in this environment by activating the virtual environment: `source venv/bin/activate`
- Then install Flask: `sudo pip install Flask`
- Test if the installation is successful and the app is running: `sudo python __init__.py` which should output if successful: `Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)`.
- Run `deactivate` to deactivate the environment.

### Configure and enable a new virtual host.
- `sudo nano /etc/apache2/sites-available/FlaskApp.conf`
- Add the following lines and save and quit:
```
<VirtualHost *:80>
		ServerName 52.31.4.176
		ServerAdmin admin@mywebsite.com
		WSGIScriptAlias / /var/www/FlaskApp/flaskapp.wsgi
		<Directory /var/www/FlaskApp/FlaskApp/>
			Order allow,deny
			Allow from all
		</Directory>
		Alias /static /var/www/FlaskApp/FlaskApp/static
		<Directory /var/www/FlaskApp/FlaskApp/static/>
			Order allow,deny
			Allow from all
		</Directory>
		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
- Now enable the virtual host:
`sudo a2ensite FlaskApp`

- Now create the wsgi file to servce the flask app.
`cd /var/www/FlaskApp
sudo nano flaskapp.wsgi` and add the following lines:
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/FlaskApp/")
from FlaskApp import app as application
application.secret_key = 'Add your secret key'
```
- Restart Apache with the following command to apply the changes:
`sudo service apache2 restart`.
- To verify you have been successful `Hello, I love Digital Ocean!` should be visible in the browser.

- Resource: [Digial Ocean](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)

## Install git.
- `sudo apt-get update`
- `sudo apt-get install git`
- `cd /var/www/`
- create a catalog directory `sudo mkdir catalog`
- move to the catalog directory `cd catalog`

## Deploy the Item Catalog project.
Clone and setup your Item Catalog project from the Github repository you created earlier in this Nanodegree program.

- `sudo git https://github.com/TheGManC/udacityServerConfig.git catalog`
- Create a wsgi file using in `/var/www/catalog` by using `sudo nano catalog.wsgi` and paste in the same example from the `FlaskApp.wsgi` example above.
- Run `sudo nano __init__.py` and add the following:
`from flask import Flask
app = Flask(__name__)
@app.route("/")
def hello():
    return "Hello, from catalog app!"
if __name__ == "__main__":
    app.run()`
- set a virtual environment by running `sudo virtualenv venv`
- Activate the virtual environment by running `source venv/bin/activate`
- Configure and enable a new virtual host.
- `sudo nano /etc/apache2/sites-available/catalog.conf`
- Add the following lines and save and quit:
`<VirtualHost *:80>
                ServerName 52.31.4.176
                ServerAdmin admin@mywebsite.com
                WSGIScriptAlias / /var/www/catalog/catalog.wsgi
                <Directory /var/www/catalog/catalog/>
                        Order allow,deny
                        Allow from all
                </Directory>
                Alias /static /var/www/catalog/catalog/static
                <Directory /var/www/catalog/catalog/static/>
                        Order allow,deny
                        Allow from all
                </Directory>
                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>`

- Next enable the `catalog` site by running `sudo a2ensite catalog` which will output the following:
`Enabling site catalog.
To activate the new configuration, you need to run:
  service apache2 reload`
- Instead run `sudo service apache2 restart`
- Visiting the site on the browser should now read `Hello, from catalog app!` if successful.
-  Make sure that your .git directory is not publicly accessible via a browser. Go to `/var/www/catalog` and run `sudo nano .htaccess` and insert the following, `RedirectMatch 404 /\.git`. Save and quit.
- Resources:  [Source,](https://stackoverflow.com/a/17916515) [Source 2](https://github.com/anumsh/Linux-Server-Configuration)

## Install Required packages
- move to `/var/www/catalog/catalog`
- activate the virtual env `source venv/bin/activate` and then install the following:
- `pip install httplib2`
- `pip install requests`
- `sudo pip install --upgrade oauth2client`
- `sudo pip install sqlalchemy`
- `pip install Flask-SQLAlchemy`
- `sudo pip install flask-seasurf`




##  Install and configure PostgreSQL:
- `sudo apt-get update`
- `sudo apt-get install postgresql postgresql-contrib`
- install the Python PostgreSQL adapter psycopg: `sudo apt-get install python-psycopg2`
- install `pip install psycopg2`
- Do not allow remote connections. Double check that remote connections are not allowed by running command `sudo nano /etc/postgresql/<version>/main/pg_hba.conf` with the version being found by running `psql -V`. Only the major and minor numbers are required in the path.
- update the `engine = create_engine()` line `dummy_data.py`, `database_setup.py` and `application.py` to read `engine = create_engine('postgresql://catalog:catalog-pw@localhost/catalog')`
- rename `application.py` to `init.py` by using `mv application.py init.py`


### Create a new database user named catalog that has limited permissions to your catalog application database.
- Change to the postgres user `sudo su - postgre` and connect `psql`
- Create the user user name catalog `CREATE USER catalog WITH PASSWORD 'catalog-pw';` and check its permissions by running `\du` which should output somthing like this:
```
List of roles
 Role name |  Attributes | Member of
-----------+-------------+-----------
 catalog   |             |  {}
```
- We will update the users permissions to allow the user to create a database: `ALTER USER catalog CREATEDB;`, now running `\du` should output something like this:

```List of roles
 Role name |  Attributes | Member of
-----------+-------------+-----------
 catalog   | Create DB   | {}
```
- now create a database named catalog `CREATE DATABASE catalog WITH OWNER catalog;`
- connect `\c catalog`
- revoke all rights and grant access to catalog
`REVOKE ALL ON SCHEMA public FROM public;` and `GRANT ALL ON SCHEMA public TO catalog;`.
- Quit connect to catalog `\q`
- Logout from postgres using `exit`.

### Now its time to run the python database scripts. 
Here, I ran into a couple of errors. I got around this by deactivating the virtual environment and leaving the catalog directory by running `cd` and give write permissions `sudo chown -R ubuntu:ubuntu catalog/`. 

- Then I went through the installed packages section again to make sure everything was set up correctly.
- If the virtual environment is not active, move to `/var/www/catalog/catalog` and run `source venv/bin/activate`.
- Now run `python database_setup.py`, and then `python dummy_data.py` to install some dummy data.
- To verify this worked, run `sudo su - postgres`, then `psql` and connect to catalog `\c catalog`, and run `select * from item;` which will output:

```
       name        | id |           description           | category_id | user_id
-------------------+----+---------------------------------+-------------+---------
 Luke Skywalker    |  1 | Dummy Data For Luke Skywalker   |           1 |       1
 Kylo Ren          |  2 | Dummy Data for Kylo Ren         |           1 |       1
 Luke's Lightsaber |  3 | Dummy Data for Lukes Lightsaber |           2 |       1
 Sand Speeder      |  4 | Dummy Data for Sand Speeder     |           3 |       1
(4 rows)
```
- Quit connect to catalog `\q`
- Logout from postgres using `exit`.

## Set it up in your server so that it functions correctly when visiting your server’s IP address in a browser.
- Restart apache `sudo service apache2 restart`
- Remove our test __init__.py from earlier `sudo rm __init__.py`
- Rename init.py to __init__.py `mv init.py __init__.py`
- Run `python __init__.py` to launch our 'Star Wars' item catalog.
- Going to the browser will inform us that there is a server error.
- Checking the error logs `cat /var/log/apache2/error.log` will reveal `IOError: [Errno 2] No such file or directory: 'client_secrets.json'`. This is because we havent sepcified the absolute path to the `client_secrets.json`. Run `sudo nano __init__.py` and update the `CLIENT_ID` to read `CLIENT_ID = json.loads(
    open(r'/var/www/catalog/catalog/client_secrets.json', 'r').read())['web']['client_id']` and also update `oauth_flow = flow_from_clientsecrets('/var/www/catalog/catalog/client_secrets.json', scope='')`
- Now restart apache by running `sudo service apache2 restart` then `python __init__.py` and check the browser again. The star wars catelog successfuly appears.

## OAUTHLogin
- Resource: [Source] (https://github.com/anumsh/Linux-Server-Configuration)

- Now go to [this site](http://www.hcidata.info/host2ip.cgi) to get the host name of your public ip address 52.31.4.176. This site tells us that our host name is `ec2-52-31-4-176.eu-west-1.compute.amazonaws.com`.
- We must now edit our `catalog.conf` by running `sudo nano /etc/apache2/sites-available/catalog.conf` and add the `ServerAlias ec2-52-31-4-176.eu-west-1.compute.amazonaws.com` below Server Admin.
- Because we are using Google Authorization we need to go to the developer console and edit our credentials. Add the hostname, `http://52.31.4.176`,  and public IP, `http://ec2-52-31-4-176.eu-west-1.compute.amazonaws.com`, to the Authorized Javascript origins.
- Add `http://ec2-52-31-4-176.eu-west-1.compute.amazonaws.com/login` and `http://ec2-52-31-4-176.eu-west-1.compute.amazonaws.com/gconnect` to the Authorized redirect URIs.

## Now Everything should work.

### Resources
- https://github.com/anumsh/Linux-Server-Configuration 
- https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps 
- https://stackoverflow.com/a/17916515) 
