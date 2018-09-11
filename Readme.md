# Linux Server

In this tutorial we're going to configure a baseline installation of Ubuntu server to host web applications on Amazon Lightsail. It is a  step by step on how to create your instance, configure the web server, and finally host a flask application securely with Transport Layer Security.


## Geting Started

### _Create the Ubuntu Linux server instance on Amazon Lightsail_

1. Login to _[Amazon Lightsail](https://lightsail.aws.amazon.com/)_. If you don't have an Amazon Web Services account, create one
1. Create an instance (server running in amazon data centers)
    a. Select the instance location (region)
    b. Pick Linux/Unix platform
    c. Select OS Only on the blueprint and then Ubuntu 16.04LTS
    d. Download the default private key from the SSH Key pair manager option so you can connect to your Lightsail instance
    e. Choose the lowest plan to get free-tier access
    f. Finally, name your instance and click create.

### _SSH into your server_

1. Copy the private key downloaded on step 2.d. of the previous section to the .ssh/ directory under the default user's home directory and modify default permissions
    a. Change to user's home directory 
    ```
    # cd ~ 
    ```
    b. Check if .ssh/ directory exists 
    ``` 
    # ls -la 
    ```
    c. If .ssh/ does not exists, create it 
    ``` 
    # mkdir .ssh 
    ```
    d. Copy the private key file to .ssh/ 
    ```
    # cp <private_key_file> ~/<user_name>/.ssh/aws-key
    ```
    e. Change permissions 
    ``` 
    # chmod 700 .ssh/
    # chmod 600 .ssh/aws-key
    ```
    
1. SSH Access to your lightsail instance
    a. Find the public IP Address of the instance
    b. Connect to the instance using the default username 
    ``` 
    # ssh ubuntu@<ip_address> -i .ssh/aws-key 
    ```
    c. Answer yes to _Are you sure you want to continue connecting (yes/no)?_

1. The ssh access to the lightsail instance is restricted to keypair authentication by default. However, it is recommend to double-check.
    ```  
    # more /etc/ssh/sshd_config |grep 'PasswordAuthentication '
    ```
    If the output is different than ```PasswordAuthentication no ```, then edit /etc/ssh/sshd_config and change it and restart the sshd daemon ``` # service sshd restart ```.

### _System updates and upgrade_
1. Update the package source list
    ```
    # sudo apt-get update
    ```
2. Upgrade your system
    ```
    # sudo apt-get upgrade
    ```

### _Firewall_

Enforce the least privelege principle by opening only the required ports.

1. Check the status of the firewall
    ```
    # sudo ufw status
    ```

1. Start the configuration blocking all incoming connections
    ```
    # sudo ufw default deny incoming
    ```

1. Allow the server's outgoing connections
    ```
    # sudo ufw default allow outgoing
    ```

1. Allow SSH connections on ports 22 and 2200
    ```
    # sudo ufw allow 22/tcp
    # sudo ufw allow 2200/tcp
    ```

1. Allow connections to http(port 80) and https(port 443)
    ```
    # sudo ufw allow www
    # sudo ufw allow https
    ```

1. Allow NTP traffic
    ```
    # sudo ufw allow 123/udp
    ```
1. On your lightsail instance page go to manage, networking, scroll down to firewall, and make sure that NTP(123/udp), SSH(22/tcp), custom ssh(2200/tcp), HTTP(80/tcp), and HTTPS(443/tcp) are allowed
    
1. To check the rules added before enabling ufw run
    ```
    # sudo ufw show added
    ```

1. Enable the firewall

    *Note: Make sure that SSH rule has been configure to allow incoming connections before enabling ufw.*
    ```
    # sudo ufw enable
    ```

#### _The Secure way to change the SSH port_

1. Edit /etc/ssh/sshd_config with nano or vi
1. Duplicate the Port paramemter and change the port of the second one to 2200
    ``` 
    Port 22
    Port 2200
    ```
1. Restart the ssh service to apply the changes
    ```
    $ sudo systemctl restart ssh.service
    ```
1. At this point you should be able to ssh using the standard and new configured port
    ```
    $ ssh ubuntu@<ip_address> -i .ssh/aws-key -p 2200
    ```
1. Delete the standard ssh port rule 
    ```
    $ sudo ufw delete allow 22/tcp
    ```
1. Comment the ```Port 22``` parameter in /etc/ssh/sshd_config
1. Restart the ssh service to apply the changes
    ```
    $ sudo systemctl restart ssh.service
    ```
1. On your lightsail instance page go to manage, networking, scroll down to firewall, edit rules, click on the X next to port 22, and save.

### _Configure Time Zone and NTP Synchronization_

1. Display the current time, date, and time zone
    ```
    $ timedatectl status
    ```

1. Set time zone to UTC
    ```
    $ timedatectl set-timezone UTC
    ```

1. Install ntp package
    ```
    # sudo apt-get install ntp
    ```
1. Edit /etc/ntp.conf, delete existing ubuntu's pool entries, and add amazon's ntp address as the preferred source and google's as a fallback
    ```
    server 169.254.169.123 prefer iburst
    server time.google.com iburst
    ```
1. Restart the NTP daemon
    ```
    # sudo service ntp restart
    ```
1. Check the synchronization status
    ```
    # ntpq -p
    ```
    
### _PostgreSQL_
1. Installation of PostgreSQL
    ```
    # sudo apt-get install postgresql
    ```
1. Check if postgresql process is running
    ```
    $ ps aux|grep postgre
    ```
1. PostgreSQL doesn't allow remote connections by default. However, it is     recommended to double-check the `/etc/postgresql/<version>/main/pg_hba.conf` file.

### _Git_
1. Check if git is installed
    ```
    $ dpkg -l | grep git
    ```

1. git Installation

    ```
    $ sudo apt-get install git
    ```

### _Apache Web Server_
1. Install the Apache Web Server
    ``` 
    # sudo apt-get install apache2
    ```

1. Check if apache process is running
    ```
    $ ps aux|grep apache
    ```
    
1. Install mod_wsgi to enable web server communication with the webapp
    ```
    $ sudo apt-get install libapache2-mod-wsgi
    ```
1. Enable mod_wsgi
    ```
    # sudo a2enmod wsgi
    ```
1. Test the WSGI module
    a. Edit ```/etc/apache2/sites-enabled/000-default.conf```
    b. Add ```WSGIScriptAlias / /var/www/html/myapp.wsgi``` right before the closing of ```</VirtualHost>```
    c. Create /var/www/html/myapp.wsgi and add this
    ```
    def application(environ, start_response):
        status = '200 OK'
        output = 'It Works!'
        response_headers = [('Content-type', 'text/plain'), ('Content-Length', str(len(output)))]
        start_response(status, response_headers)
        return [output]
    ```
    d. Restart the web server
    ```
    # sudo service apache2 restart
    ```
    e. Browse to ```http://<public_IP>``` and you will see "It Works!".

1. To secure apache a little bit more, edit /etc/apache2/conf-enabled/security.conf with the following instructions to prevent
    a. Giving the OS information ```ServerSignature Off```
    b. Direct access to .sql, .git, .json, .md files
    ```
    <DirectoryMatch "/\.git">
        Require all denied
    </DirectoryMatch>

    <FilesMatch "\.(json)$">
        Require all denied
    </FilesMatch>

    <FilesMatch "\.(sql)$">
        Require all denied
    </FilesMatch>

    <FilesMatch "\.(md)$">
        Require all denied
    </FilesMatch>
    ```
    c. Embedding and Sniffing
    ```
    Header set X-Content-Type-Options: "nosniff"
    Header set X-Frame-Options: "sameorigin"
    ```
1. Edit /etc/apache2/apache2.conf to disable Directory Browsing
    ```
    <Directory /var/www/>
        Options -Indexes
        Options FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>
    ```
    In addition, make sure that apache doesn't allow access to the root filesystem outside of /var/www/
    ```
    <Directory />
        Options FollowSymLinks
        AllowOverride None
        Require all denied
    </Directory>
    
    <Directory /usr/share>
        AllowOverride None
        Require all denied 
    </Directory>
    ```
1. Enable mod_headers
    ```
    # sudo a2enmod headers
    ```
1. Restart Apache
    ```
    # sudo service apache2 restart
    ```
    
### Create the SSL/TLS certificate

Lets create a free certificate from letsencrypt.org.
1. Install certbot to fetch a certificate from Let's Encrypt
    ```
    $ sudo apt-get install software-properties-common
    $ sudo add-apt-repository ppa:certbot/certbot
    $ sudo apt-get update
    $ sudo apt-get install python-certbot-apache 
    ```
1. Create the (A) record on your DNS hosting provider; itemcatalog.wo984c.net will be used on this example
1. Configure the virtual host for itemcatalog.wo984c.net by adding the following lines to /etc/apache2/sites-available/itemCatalog.conf, remember to use sudo
    ```
    <VirtualHost *:80>
        ServerName itemcatalog.wo984c.net
        ServerAdmin wo984c@me.com
        WSGIScriptAlias / /var/www/html/myapp.wsgi
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>
    ```
1. Enable itemCatalog site and restart apache
    ```
    # sudo a2ensite itemCatalog
    # sudo service apache2 restart
    ```    
1. Create the LetsEncrypt free certificate
    ```
    # sudo certbot --apache
    ```
    Running this command will get a certificate for you and have Certbot edit your Apache configuration automatically to serve it
    
1. Go to https://itemcatalog.wo984c.net to test the secure site.

### _Prepare the environment to host our Flask Web App_
1. Install the package management system that will be used to install and manage software packages written in Python
    ```
    # sudo apt-get install python-pip
    ```
1. Virtual Environment will be used to keep the application and its dependencies isolated from the main system
    ```
    # sudo -H pip install virtualenv
    ```
1. Create the applications folders. In this case, the folder will be created in /var/www/html/
    ```
    # sudo mkdir /var/www/html/apps
    ```
1. Clone the Item Catalog web app from GitHub
    ```
    # cd /var/www/html/
    # sudo chown -R ubuntu apps/
    # sudo chgrp -R ubuntu apps/
    # git clone https://github.com/wo984c/itemCatalog.git
    ```
1. Change dir to itemCatalog and create the virtual environment for the itemCatalog project
    ```
    # cd itemCatalog/
    # virtualenv venv
    ```
1. Activate the virtual environment for the itemCatalog project
    ```
    # source venv/bin/activate
    ```
1. Install the python packages required by the Item Catalog Flask app
    ```
    (venv)# pip install Flask flask-sqlalchemy oauth2client psycopg2-binary Flask-HTTPAuth html5lib urllib3 requests redis pyparsing passlib packaging
    ```
1. Deactivate the virtual environment and go back to the system’s default Python interpreter with all its installed libraries
    ```
    (venv)# deactivate
    ```
1. Create the wsgi file used by apache to serve the Item Catalog flask app
    ```
    # cd ..
    # pwd
    /var/www/html/apps
    # vi itemCatalog.wsgi
    ```
    add the following code and save the file
    ```
    #!/usr/bin/python
    import sys
    import logging

    activate_this = '/var/www/html/apps/itemCatalog/venv/bin/activate_this.py'
    execfile(activate_this, dict(__file__=activate_this))

    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0,"/var/www/html/apps/")

    from itemCatalog import app as application
    application.secret_key = '23d34f@34#$%^fgty***2'
    ```
1. Edit the ssl version of the itemCatalog configuration in /etc/apache2/sites-available/, remember to use sudo
    ```
    <IfModule mod_ssl.c>
    <VirtualHost *:443>
        ServerName itemcatalog.wo984c.net 
        ServerAdmin wo984c@me.com
        #Location of the items-catalog WSGI file
        WSGIScriptAlias / /var/www/html/apps/itemCatalog.wsgi
        #Allow Apache to serve the WSGI app from our catalog directory
        <Directory /var/www/html/apps/itemCatalog/>
            WSGIProcessGroup itemCatalog
            WSGIApplicationGroup %{GLOBAL}
            Require all granted
            <FILES itemCatalog.wsgi>
                Require all granted
            </FILES>
        </Directory>

        #Allow Apache to deploy static content
        <Directory /var/www/html/apps/itemCatalog/static>
            Require all granted
        </Directory>

        <Directory /var/www/html/apps/itemCatalog/templates>
            Require all granted
        </Directory>

        <Directory /var/www/html/apps/itemCatalog/handlers>
            Require all granted
        </Directory>

        <Directory /var/www/html/apps/itemCatalog/models>
            Require all granted
        </Directory>
     
        <Directory /var/www/html/apps/itemCatalog/helpers>
            Require all granted
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined

    SSLCertificateFile /etc/letsencrypt/live/itemcatalog.wo984c.net/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/itemcatalog.wo984c.net/privkey.pem
    Include /etc/letsencrypt/options-ssl-apache.conf
    </VirtualHost>
    </IfModule>
    ```

1. Required Modifications to original code
    a. Oauth parameters in ``` handlers/online_oauth.py ```
    ```
    CLIENT_ID = json.loads(
    open('/var/www/html/apps/itemCatalog/client_secrets.json','r').read())['web']['client_id']
    ```
    ```
    oauth_flow = flow_from_clientsecrets('/var/www/html/apps/itemCatalog/client_secrets.json', scope='')
    ```
    ```
    app_id = json.loads(open('/var/www/html/apps/itemCatalog/fb_client_secrets.json', 'r').read())['web']['app_id']
    ```
    ```
    app_secret = json.loads(open('/var/www/html/apps/itemCatalog/fb_client_secrets.json', 'r').read())['web']['app_secret']
    ```
    b. ``` handlers/category.py ```
    ```
    from ..models.category import *
    from ..models.item import *
    from ..helpers import *
    ```
    c. ``` handlers/endpoint.py ```
    ```
    from ..models.item import Item
    from ..models.category import Category
    from ..helpers import *
    ```
    d. ``` handlers/item.py ```
    ```
    from ..models.category import *
    from ..models.item import *
    from ..helpers import *
    ```
    e. ``` __init__.py ```
    ```
    from .helpers import *
    ```
1. Change owner and group of /var/www/html/apps to www-data
    ```
    # cd /var/www/html
    # sudo chown -R www-data apps/
    # sudo chgrp -R www-data apps/
    ```
1. Restart Apache Web Server
    ```
    # sudo service apache2 restart
    ```

### _Give grader SSH Access_

1. Generate SSH Key Pair for grader on the client system
    ```
    # cd ~
    # mkdir grader_keys
    # ssh-keygen -f grader_keys/key -t rsa
    ```
    Enter and re-enter a passphrase when prompted.

1. Create grader user in the lightsail instance
    ```
    $ sudo adduser grader --disabled-password
    ```
    Enter the information requested (Full name, phone, etc) and answer Y to _Is the information correct? [Y/n]_

1. To check if user has been created correctlly 
    ```
    $ finger grader 
    ```
    Note: Lightsail Ubuntu image doesn't has finger installed by default. You can install it by typing 
    ```
    $ sudo apt install finger
    ```

1. Change directory to grader's home and create the .ssh directory. This directory will be used to store the public key file
    ```
    # sudo su -
    # cd ~grader
    # mkdir .ssh
    # chmod 700 .ssh/
    ```
1. Now copy the content of ~/grader_keys/key.pub file on the client and paste it to /home/grader/.ssh/authorized_keys on the server.

1. Make sure that authorized_keys file is owned by user/group grader and permissions are set to -rw-------
    ```
    # chmod 600 authorized_keys
    # cd ..
    # chown -R grader .ssh/
    # chgrp -R grader .ssh/
    ```
1. Give Sudo Access to grader
    ```
    $ visudo -f /etc/sudoers.d/udacity-grader
    ```
    Add the following line to the file and save it
    ```
    grader  ALL=(ALL)   NOPASSWD:ALL
    ```
1. Finally, share the private key located in the client's ~/grader_keys/ dir with the Udacity's grader.
