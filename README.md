# Linux_server_configuration
This is the final project for Udacity's [Full Stack Web Developer Nanodegree](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004). 

This page explains how to secure and set up a Linux distribution on a virtual machine, install and configure a web and database server to host a web application. 
- The Linux distribution is [Ubuntu](https://www.ubuntu.com/download/server) 16.04 LTS.
- The virtual private server is [Amazon](https://lightsail.aws.amazon.com/).
- The web application is my [Item Catalog project](https://github.com/boisalai/udacity-catalog-app) created earlier in this Nanodegree program.
- The database server is [PostgreSQL](https://www.postgresql.org/).

You can visit http://54.164.136.189/director  or http://ec2-54-164-136-189.compute-1.amazonaws.com/director for the website deployed.

## Get a server
### Step 1: Start a new Ubuntu Linux server instance on Amazon EC2 

  - Login to *[aws.amazon.com](https://console.aws.amazon.com)* and login to default user (ubuntu)
  - Choose EC2 and Launch Instance with appropriate settings.
  - Check for instance IPv4 public IP - 54.164.136.189
  - we can download a .pem file and connect with following command
    ```
    ssh -i LinuxServer.pem ubuntu@54.164.136.189
    ``` 
  - 22 is Port by Default,Later we need to change it to 2200 as per the
    udacity-linux-server-configuration rubrics.

## Secure the server

### Step 2: Update and upgrade installed packages

```
sudo apt-get update
sudo apt-get upgrade
```


### Step 3: Change the SSH port from 22 to 2200

- Edit the `/etc/ssh/sshd_config` file: `sudo vi /etc/ssh/sshd_config`.
- Change the port number on line 5 from `22` to `2200`.
- Save and exit using esc and confirm with :wq.
- Restart SSH: `sudo service ssh restart`.
- Change inbound rules in Amazon EC2 --> Type : Custom TCP Rule as 2200
- To check port 2200 wether working or not by `ssh -i LinuxServer.pem -p 2200 ubuntu@54.164.136.189` 

### Step 4: Configure the Uncomplicated Firewall (UFW)

- Configure the default firewall for Ubuntu to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).
  ```
  sudo ufw status                  # The UFW should be inactive.
  sudo ufw default deny incoming   # Deny any incoming traffic.
  sudo ufw default allow outgoing  # Enable outgoing traffic.
  sudo ufw allow 2200/tcp          # Allow incoming tcp packets on port 2200.
  sudo ufw allow www               # Allow HTTP traffic in.
  sudo ufw allow 123/udp           # Allow incoming udp packets on port 123.
  sudo ufw deny 22                 # Deny tcp and udp packets on port 53.
  ```

- Turn UFW on: `sudo ufw enable`. The output should be like this:
  ```
  Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
  Firewall is active and enabled on system startup
  ```

- Check the status of UFW to list current roles: `sudo ufw status`. The output should be like this:

  ```
  Status: active
  
  To                         Action      From
  --                         ------      ----
  2200/tcp                   ALLOW       Anywhere                  
  80/tcp                     ALLOW       Anywhere                  
  123/udp                    ALLOW       Anywhere                  
  22                         DENY        Anywhere                  
  2200/tcp (v6)              ALLOW       Anywhere (v6)             
  80/tcp (v6)                ALLOW       Anywhere (v6)             
  123/udp (v6)               ALLOW       Anywhere (v6)             
  22 (v6)                    DENY        Anywhere (v6)
  ```


## Give `grader` access

### Step 5: Create a new user account named `grader`

- While logged in as `ubuntu`, add user: `sudo adduser grader`. 
- Enter a password (twice) and fill out information for this new user.


### Step 6: Give `grader` the permission to sudo

- Edits the sudoers file: `sudo visudo`.
- Search for the line that looks like this:
  ```
  root    ALL=(ALL:ALL) ALL
  ```

- Below this line, add a new line to give sudo privileges to `grader` user.
  ```
  root    ALL=(ALL:ALL) ALL
  grader  ALL=(ALL:ALL) ALL
  ```

- Save and exit using CTRL+X and confirm with Y.
- Verify that `grader` has sudo permissions. Run `su - grader`, enter the password.


### Step 7: Create an SSH key pair for `grader`

  -Configure key-based authentication for grader user
  - create .ssh folder by `mkdir /home/grader/.ssh`
  - Run this command `cp /home/ubuntu/.ssh/authorized_keys /home/grader/.ssh/authorized_keys`
  - change ownership `chown grader.grader /home/grader/.ssh`
  - add 'grader' to sudo group `usermod -aG sudo grader`
  - change permissions for .ssh folder `chmod 0700 /home/grader/.ssh/`, for authorized_keys `chmod 644 authorized_keys`
  - Check in `vi /etc/ssh/sshd_config` file if `PermitRootLogin` is set to `no`
  - Change to 'nano /etc/ssh/sshd_config' 
  - Change 'PasswordAuthentication' from 'no' to 'yes'.
  - Add 'AllowUsers grader'  at end of the file so that we will login through grader.
  - Restart SSH: `sudo service ssh restart`
  - On the local machine, checking if the grader account working or not by running this command : `ssh -i LinuxServer.pem -p 2200 grader@54.164.136.189`.
  - If the above command shows error as Permission Denied then connect with `ssh -i LinuxServer.pem -p 2200 ubuntu@54.164.136.189` 
  and then login with `su - grader`
  
## Prepare to deploy the project

### Step 8: Configure the local timezone to UTC

- While logged in as `grader`, configure the time zone: `sudo dpkg-reconfigure tzdata`. Choose time zone UTC.

### Step 9: Install and configure Apache to serve a Python mod_wsgi application

- While logged in as `grader`, install Apache: `sudo apt-get install apache2`.
- Enter public IP of the Amazon EC2 instance into browser. Check Apache is working or not by executing public IP.
- My project is built with Python 3. So, I need to install the Python 3 mod_wsgi package:  
 `sudo apt-get install libapache2-mod-wsgi-py3`.
- Enable `mod_wsgi` using: `sudo a2enmod wsgi`.


### Step 10: Install and configure PostgreSQL
  - `sudo apt-get install libpq-dev python-dev`
  - `sudo apt-get install postgresql postgresql-contrib`
  - `sudo su - postgres`
  - `psql`
  - `CREATE USER catalog WITH PASSWORD 'catalog';`
  - `ALTER USER catalog CREATEDB;`
  - `CREATE DATABASE catalog WITH OWNER catalog;`
  - `\c catalog`
  - `REVOKE ALL ON SCHEMA public FROM public;`
  - `GRANT ALL ON SCHEMA public TO catalog;`
  - `\q`
  - `exit`
  - Switch back to the `grader` user: `exit`.

### Step 11: Install git

- While logged in as `grader`, install `git`: `sudo apt-get install git`.

## Deploy the Item Catalog project

### Step 12.1: Clone and setup the Item Catalog project from the GitHub repository 

- While logged in as `grader`,
- From the `/var/www` directory, Clone the catalog project:<br>
`sudo git clone https://github.com/ikkurthipoojitha/catalog.git`.
- Change the ownership of the `catalog` directory to `grader` using: `sudo chown -R grader:grader catalog/`.
- Change to the `/var/www/catalog/catalog` directory.
- Rename the `movie.py` file to `__init__.py` using: `mv movie.py __init__.py`.
- We need to change sqlite to postgresql create_engine in `__init__.py`,`movie_database.py` and `movie_info.py`,
   ```
   # engine = create_engine("sqlite:///catalog.db")
   engine = create_engine('postgresql://catalog:catalog@localhost/catalog')
   ``` 

### Step 12.2: Authenticate login through Google

- Go to [Google Cloud Plateform](https://console.cloud.google.com/).
- Click `APIs & services` on left menu.
- Click `Credentials`.
- Create an OAuth Client ID (under the Credentials tab), and add http://54.164.136.189.xip.io and
http://ec2-54-164-136-189.compute-1.amazonaws.com/ as authorized JavaScript 
origins.
- Add http://54.164.136.189.xip.io/login,http://54.164.136.189.xip.io/gconnect,http://54.164.136.189.xip.io/callback, http://54.164.136.189.xip.io/director
as authorized redirect URI.
- Download the corresponding JSON file, open it and copy the contents.
- Open `/var/www/catalog/catalog/client_secret.json` and paste the previous contents into the this file.
- Replace the client ID `templates/login.html` file in the project directory.


### Step 13.1: Install the virtual environment and dependencies

- While logged in as `grader`, install pip: `sudo apt-get install python3-pip`.
- Install the virtual environment: `sudo apt-get install python-virtualenv`
- Change to the `/var/www/catalog/catalog/` directory.
- Create the virtual environment: `sudo virtualenv -p python3 venv3`.
- Change the ownership to `grader` with: `sudo chown -R grader:grader venv3/`.
- Activate the new environment: `. venv3/bin/activate`.
- Install the following dependencies:
  ```
  pip install httplib2
  sudo apt-get 
  pip install requests
  pip install --upgrade oauth2client
  pip install sqlalchemy
  pip install flask
  sudo apt-get install libpq-dev
  pip install psycopg2-binary
  ```


### Step 13.2: Set up and enable a virtual host

  Configure and enable a new virtual host
  - Run this: `sudo vi /etc/apache2/sites-available/catalog.conf`
  - Paste this code: 
  ```
  <VirtualHost *:80>
      ServerName 54.164.136.189.xip.io
      ServerAlias ec2-54-164-136-189.compute-1.amazonaws.com
      ServerAdmin ubuntu@54.164.136.189
      WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/catalog/venv3/lib/python3.5/site-packages
      WSGIProcessGroup catalog
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
  </VirtualHost>
  ```
  - Enable the virtual host `sudo a2ensite catalog`

  Enabling site catalog.
  To activate the new configuration, you need to run:
    service apache2 reload
  
  - Reload Apache: `sudo service apache2 reload`.

### Step 13.3: Set up the Flask application

- Create `/var/www/catalog/catalog.wsgi` file add the following lines:

  ```
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0, "/var/www/catalog/")
    from catalog import app as application
    application.secret_key = 'supersecretkey'
  ```
- Restart Apache: `sudo service apache2 restart`.
- From the `/var/www/catalog/catalog/` directory, 
  activate the virtual environment: `. venv3/bin/activate`.
- Run: `python movie_database.py`.
- Deactivate the virtual environment: `deactivate`.

### Step 13.4: Disable the default Apache site

- Disable the default Apache site: `sudo a2dissite 000-default.conf`. 
The following prompt will be returned:

  ```
  Site 000-default disabled.
  To activate the new configuration, you need to run:
    service apache2 reload
  ```

- Reload Apache: `sudo service apache2 reload`.

### Step 13.5: Launch the Web Application

- Restart Apache again: `sudo service apache2 restart`.
- Open your browser to http://54.164.136.189/director  or http://ec2-54-164-136-189.compute-1.amazonaws.com/director

### Step 14: Automatically install updates(optional)

The `unattended-upgrades` package can be used to automatically install important system updates.
- Enable automatic (security) updates: `sudo apt-get install unattended-upgrades`.
- Edit `/etc/apt/apt.conf.d/50unattended-upgrades`, uncomment the line `${distro_id}:${distro_codename}-updates` and save it.
- Modify `/etc/apt/apt.conf.d/20auto-upgrades` file so that the upgrades are downloaded and installed every day:
  ```
  APT::Periodic::Update-Package-Lists "1";
  APT::Periodic::Download-Upgradeable-Packages "1";
  APT::Periodic::AutocleanInterval "7";
  APT::Periodic::Unattended-Upgrade "1";
  ```
- Enable it: `sudo dpkg-reconfigure --priority=low unattended-upgrades`.
- Restart Apache: `sudo service apache2 restart`.

## Useful commands

 - To get log messages from Apache server: `sudo tail /var/log/apache2/error.log`.
 - To restart Apache: `sudo service apache2 restart`.

## Other Helpful Resources

- DigitalOcean [How To Deploy a Flask Application on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
- GitHub Repositories 
  - [SkBadulla/LinuxServerConfiguratio](https://github.com/SkBadulla/LinuxServerConfiguration)

