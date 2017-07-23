# Linux-server-configuration
### Project 
 
Build a Virtual Machine on the Tencent Cloud Platform and configure a web application on this server.

+ VM: Ubuntu 16.04 LTS
+ IP : 182.254.131.23
+ HTTP Port: 80
+ SSH Port: 2200
+ NTP Port: 123

The reason why I use Tencent Cloud than AWS is that I don't have a credit card. However, It still works fine for sever-configuration.

### Configure Steps

**1. SSH to login server**

+ Create the ssh key on server named ubuntu_instance. '

+ Download the private ssh key
+ Give the right permission  ```chmod 600 ubuntu_instance``` 

+ SSH into the server   ``` ssh -i ubuntu_instance ubuntu@182.254.131.23``` 

PS. Tencent Cloud assigned username is ubuntu not root. Because we finally only login in as grader, we can treat ubuntu as root in this case. 

**2. Update installed packages**

+ Update download list  ```sudo apt-get update```

+ Upgrade package ```sudo apt-get upgrade```


**3. Change ssh port to 2200**

+ Edit file use ```sudo vim /etc/ssh/sshd_config```

+ Change the port 22 to 2200 and save it

+ Restart ssh ```service ssh restart```.

+ Relogin use port 2200 ```ssh -i ubuntu_instance -p 2200 ubuntu@182.254.131.23```

+ Still use port 22 would get error ```ssh: connect to host 182.254.131.23 port 22: Connection refused``` 

**4. Create user grader**

+ Run ```sudo adduser grader```, and set password 12345678.

+ Run ```sudo chmod 775 /etc/sudoers```

+ Open file ```sudo vim /etc/sudoers```

+ Add ```grader ALL=(ALL:ALL) NOPASSWD: ALL``` on the last.

**5. Configure grader to ssh server**

+ Create diretory  ```sudo mkdir  /home/grader/.ssh```

+ Copy authentication key to grader``` cp /home/ubuntu/.ssh/authorized_keys /home/grader/.ssh/authorized_keys ```

+ Restart ssh  ```service ssh restart```

+ Relogin as grader```ssh -i ubuntu_instance -p 2200 grader@182.254.131.23```

**6. Disable root  and ubuntu ssh login**

+ Run ```sudo vim /etc/ssh/sshd_config```

+ Change ```PermitRootLogin prohibit-password``` to ```PermitRootLogin no``` 

+ Delete user ubuntu or only delete its authentication key.

+ Now the only to login is using ```ssh -i ubuntu_instance -p 2200 grader@182.254.131.23```

**7. Configure the Uncomplicated Firewall (UFW)**

+ Set SSH ```sudo ufw allow 2200/tcp```

+ Set HTTP ```sudo ufw allow 80/tcp```

+ Set NTP ```sudo ufw allow 123/udp```

+ Enable UFW ```sudo ufw enable```

**8. Change timezone to UTC**

+ Execute ```sudo dpkg-reconfigure tzdata```

+ Scroll to the bottom of the Continents list and select None of the above; in the second list, select UTC.

**9. Install Apache**

+ ```sudo apt install apache2```

**10.  Install mod_wsgi**

+ Install```sudo apt install libapache2-mod-wsgi python-dev```

+  Enable wod_wsgi by ```sudo a2enmod wsgi```

+  Start apache service ```sudo service apache2 start```

**11. Git clone catalog app**

+ Install git ``` sudo apt install git ```

+ Create diretory for app ```sudo mkdir /var/www/catalog```

+ Change the owner of catalog folder ```sudo chown -R grader:grader catalog ```

+ Get into the directory ```cd /var/www/catalog```

+ Use http clone ```git clone https://github.com/kaka2634/Catalog.git```

**12. Change the project file name and content**

+ Rename project.py to __init__.py ```mv project.py __init__.py```

+ Open the __init.py__ ```vim __init.py__```

+ Change all the client_secrets.json to /var/www/catalog/catalog/client_secrets.json

**13. Install python-pip and Flask and other dependency**

```
$  sudo apt-get install python-pip
$  pip install Flask
$  sudo pip install httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils
$  sudo pip install requests
```
**14. Create catalog.wsgi**

In directory /var/www/catalog create file ```/var/www/catalog$ vim catalog.wsgi```

Add following content to the file and save

```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from Catalog import app as application
application.secret_key = 'supersecretkey'
```

**15. Install Virtual environment**

+ Install the virtual environment ```sudo pip install virtualenv```

+ Create a new virtual environment with ```sudo virtualenv venv```
    
 + Activate the virutal environment ```source venv/bin/activate```
 
 + Change permissions ```sudo chmod -R 777 venv```
 
 + Now you can see the venv folder under Catalog folder.
 
**16 Configure virtual host**

+ Edit file ```sudo vim /etc/apache2/sites-available/catalog.conf```

+ Add following content to the file

```
<VirtualHost *:80>
    ServerName 182.254.131.23
    ServerAdmin admin@182.254.131.23
    WSGIDaemonProcess catalog python-path=/var/www/catalog/Catalog:/var/www/catalog/Catalog/venv/lib/python2.7/site-packages
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/Catalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/Catalog/static
    <Directory /var/www/catalog/Catalog/static>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

```

+ Enable the virtual host ```sudo a2ensite catalog```

**17. Install and configure PostgreSQL**

+ Run following comands in order. 

```
$  sudo apt-get install libpq-dev python-dev
$  sudo apt-get install postgresql postgresql-contrib
$  sudo su - postgres
$  psql
$  CREATE USER catalog WITH PASSWORD 'password';
$  ALTER USER catalog CREATEDB;
$  CREATE DATABASE catalog WITH OWNER catalog;
$  \c catalog
$  REVOKE ALL ON SCHEMA public FROM public;
$ GRANT ALL ON SCHEMA public TO catalog;
$  \q
$  exit
```

+ Change create engine line in  __init__.py, category_database_setup.py and insert_data_database.py to ```engine = create_engine('postgresql://catalog:password@localhost/catalog')```

+ Create database by run python code
```
     python category_database_setup.py
     python  insert_data_database.py 
     
 ```
 
 + Make sure no remote connections to the database are allowed.  Edit file ```sudo vim /etc/postgresql/9.3/main/pg_hba.conf```, and it should like 
  
 ```
local   all             postgres                                peer
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md
 
 ```
 
**18. Restart Apache**

+ ``` sudo service apache2 restart```
 
**19. Error fix**

+ View error log ```sudo vim /var/log/apache2/error.log``` 

+ Use Shift+g to get bottom

+ When meet error```ERROR  open('client_secrets.json', 'r').read())['web']['client_id'] ``` refer to section 12 to fix it

+ When meet error ```ImportError: No module named catalog``` change the catalog to Catalog fix it

+ When meet error ``` raise RuntimeError('The session is unavailable because no secret ``` refer to section 14 add secret key to fix it

 
**20. Visit page site at [http://182.254.131.23](http://182.254.131.23)**

Thank for [rrjoson](https://github.com/rrjoson/udacity-linux-server-configuration) best README helping me a lot.

PS. Login function is not working because I have add origin http://182.254.131.23 to google app api. This function would be done in furture. If you know how to do this please help me. Google said it need .com address rather than ip addresss.