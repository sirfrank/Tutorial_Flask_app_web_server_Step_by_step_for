# resources :
##### setup txt file :
[a link](https://github.com/sirfrank/Tutorial_Flask_app_setup_bash_script/blob/main/setup.txt)

##### more debugging:
[a link](https://github.com/sirfrank/Flask_app)
# Guide to run your app on localhost

``` cd <your_app_directory>```

``` source <yourenv>/bin/activate```	# if you are using envoirment, as you SHOULD !

``` export FLASK_APP=run.py ```

``` export FLASK_DEBUG=1 ```

``` flask run --host=0.0.0.0 ``` # this will run your app localhost:5000




# Step by step guide to host Flask application on webserver

####################################################################################################
#####								this guide will take you throu the process of 				#####
#####								setting up a webserver to serve Flask application			#####
#####								on a Centos 8 VPS											#####
####################################################################################################

# STEP 0  (requirements for flask web deployment)

## options

### Rpi for local storadge
possible to run on Raspberry pi, for local network acces is fine, but if public access needed, some networking must be done (IP allocation, static IP, etc)

### served app on managed platforms 
- heroku.com
- pythonanywhere.com
- etc

### self managed VPS
Step by step guide in this tutorial


# STEP 1 (initial server setup)

## Vultr.com (register if needed)
https://www.vultr.com/?ref=7757848 # if registered via this link , in theory u get $100 credit. (never tested, but should work)

- deploy new server

### server options:
- Choose Server: Cloud Compute
- Server Location : Frankfurt (or the closest for your audience location)
- Server Type : Centos 8
- Server Size : $5/mo package:  25 GB SSD, 1 CPU 1024MB Memory, 1000GB Bandwidth (enough for basic Flask)
- Click on deploy now

in 5-10 mins, your server is deployed.




# STEP 2 (user management)

### connect to server via ssh

click on the deployed server on Vultr.com, you will find yourIP address, and root user password there.

open up terminal :

- Windows : use PUTTY
- Linux : use terminal (Ctr+Alt T)

connect to the server:
``` ssh root@<yourIP> ```
	than type root password

### create Flask_user user
- add user
``` adduser Flask_user ```
- add passwd
``` passwd Flask_user ```

- add to wheel group, which is sudoer
```usermod -aG wheel Flask_user```

- quit server and login back as Flask_user user

quit root user ``` exit ```

# STEP 3 (install packages for Centos8)

log back to server with Flask_user user in terminal/PUTTY ``` ssh Flask_user@<yourIP> ```

update centos8 to latest ``` sudo yum clean all && yum clean metadata && yum -y update yum \ && yum update --skip-broken -y; ```
this might take a while depending on how updated your initial Centos

### epel + nano + nginx

install nano for text editing (yeah, i know... vim.. but if u are reading this tutorial, you might be more confortable with nano)
``` yum install epel-release -y; yum install nano -y ```

potential error :

“Failed to set locale, defaulting to C.UTF-8” 

Fix: ```  localectl set-locale LANG=en_US.UTF-8 ``` 

if still exist :

```dnf install langpacks-en glibc-all-langpacks -y```



install nginx for web-server
``` yum install nginx -y; ```

set your server timezone ``` timedatectl set-timezone Europe/Budapest; ``` (or whatever is your timezone)


make sure all updated ``` yum update -y; ```

enable nginx ``` sudo systemctl enable nginx ```

### firewall settings

 open port 5000 for testing ```sudo firewall-cmd --zone=public --add-port=5000/tcp --permanent ```

 open port 8000 for testing ```sudo firewall-cmd --zone=public --add-port=8000/tcp --permanent ```
 
 add http : ```sudo firewall-cmd --permanent --add-service=http```

 reload firewall to make effect the above ``` sudo firewall-cmd --reload ```

 ### turn off selinux


 turn off selinux ``` sudo nano /etc/sysconfig/selinux ```
 
 Change the directive SELinux=enforcing to SELinux=disabled

 press Ctrl+X ; press y 	# save the file

 

 

# STEP 4 (checking if all ok for now)

reboot your server ``` sudo reboot ``` it will take few mins

log back to server with Flask_user user in terminal/PUTTY ``` ssh Flask_user@<yourIP> ```

### security (firewall + selinux)
``` sudo firewall-cmd --list-ports --zone=public ``` # should see ports 5000 and 8000 open

``` sestatus ``` # should shows that selinux is disabled


###  nginx
check if running ``` sudo systemctl status nginx ```

web browser : check your IP address, should shows a default nginx page

# STEP 5 (app setup)
### create folder for application
```mkdir ~/Flask_app;```
```cd /;```
```chmod 770 Flask_user/;```
```cd /home/Flask_user/Flask_app;```
######### you must be in /home/Flask_user/Flask_app;


### virtualenv
- this is optional THOU very recommended for later, as python and it dependecies/libraries has a zillion of versions, and not all of them are compatible with other versions ! Version controll is very important in python codes !

install virtualenv ``` sudo pip3 install virtualenv ```

better : ```pip3 install --user Flask_user virtualenv```

if "egg_info" error, su root, and do as root. 


create virtual env for this project ```python3 -m virtualenv env_Flask```

activate virtual env ```source env_Flask/bin/activate```


### install flask + gunicorn

The Gunicorn "Green Unicorn" is a Python Web Server Gateway Interface HTTP server

``` pip3 install gunicorn flask ```

### create run.py

``` sudo nano ~/Flask_app/run.py ```

insert the below code

```
from flask import Flask
application = Flask(__name__)

@application.route("/")
def hello():
    return "<h1 style='color:blue'>Hello There!</h1>"

if __name__ == "__main__":
    application.run(host='0.0.0.0')
```

press Ctrl+X ; press y 	# save the file

### create wsgi

```sudo nano ~/Flask_app/wsgi.py```

insert the below code

```
from run import application

if __name__ == "__main__":
    application.run()
```
press Ctrl+X ; press y 	# save the file

# STEP 6 (test app)

## test 1
```cd ~/Flask_app/```  	# move to app folder

```python3 run.py```		# run run.py

It should start your web app on yourPI:5000

open web browser, and check yourPI:5000

## test 2

``` gunicorn --bind 0.0.0.0:8000 wsgi ```

It should start your web app on yourPI:8000

open web browser, and check yourPI:8000


## deactivate virtualenv

``` deactivate ```

# STEP 7 (setup Flask_app service)

This section creates a system file, which will tell your system to boot your Flask_app with system boot

``` sudo nano /etc/systemd/system/Flask_app.service ```

insert the below code
```
[Unit]
Description=Gunicorn instance to serve Flask_app
After=network.target

[Service]
User=Flask_user
Group=nginx
WorkingDirectory=/home/Flask_user/Flask_app
Environment="PATH=/home/Flask_user/Flask_app/env_Flask/bin"
ExecStart=/home/Flask_user/Flask_app/env_Flask/bin/gunicorn --workers 3 --bind unix:Flask_app.sock -m 007 wsgi

[Install]
WantedBy=multi-user.target
```
press Ctrl+X ; press y 	# save the file

``` sudo systemctl start Flask_app```		# start service

```sudo systemctl enable Flask_app```		# enable service to start with system boot

# STEP 8 (configure nginx to serve )

``` sudo nano /etc/nginx/nginx.conf   ```

above the other server {} block 

insert the below code  (change <yourIP> at server_name)

```
server {
    listen 80;
    server_name <yourIP>;

    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass http://unix:/home/Flask_user/Flask_app/Flask_app.sock;
    }
}
```
press Ctrl+X ; press y 	# save the file

add nginx user to Flask_user group
``` sudo usermod -a -G Flask_user nginx ```


check if all ok with nginx
``` sudo nginx -t ``` 

enable nginx as a service

``` sudo systemctl start nginx ``` 


```  sudo systemctl enable nginx ``` 

# STEP 9 (Now big test)

### reboot server 
close not needed ports 

``` sudo firewall-cmd --zone=public --remove-port=5000/tcp ``` 

``` sudo firewall-cmd --zone=public --remove-port=8000/tcp ``` 

``` sudo firewall-cmd --runtime-to-permanent ``` 

``` sudo firewall-cmd --reload ``` 

reboot the server

``` sudo reboot ``` 

### If all foes ok, go to Step 10...

after reboot your server should serve the Flask_app

visit yourIP in webbrowser

### OR 

# Debugging :)(listed cases i f*cked up)

ERROR : ``` This site can’t be reachedhttp://<yourIP>/ is unreachable.```

- check if nginx is running :
```service nginx status```
if not, service nginx restart

- check if Flask_app service is running
```service Flask_app status```

## If failed with Flask_app service code :
```Process: 715 ExecStart=/home/Flask_user/Flask_app/env_Flask/bin/gunicorn --workers 3 --bind unix:Flask_app.sock -m 007 wsgi (code=exited, status=217/USER)```

Something fucked upwith user craetion .... me idiot :D

## If failed with Flask_app service code :
```Process: 727 ExecStart=/home/Flask_user/Flask_app/env_Flask/bin/gunicorn --workers 3 --bind unix:Flask_app.sock -m 007 wsgi (code=exited, status=3/NOTIMPLEMENTED)```

doing it right now :D
If failed with code :
```any new error code```
- check if gunicorn is in env_folder: 
```/home/Flask_user/Flask_app/env_Flask/bin/```in this folder must be gunicorn. (otherwise pip3 install failed.)
- check if gunicorn is running on its own
```/home/Flask_user/Flask_app/env_Flask/bin/gunicorn --workers 3 --bind=0.0.0.0:8000 wsgi```


more debugging trick : [a link](https://github.com/sirfrank/Flask_app)


# Step 10 (setting up wsgi.py -> my_flask_app_package.__init__.py -> routes.py )

## create python package for your modul
``` cd /home/Flask_user/Flask_app ```

``` mkdir my_flask_app_package ``` 

``` cd my_flask_app_package ```

``` nano __init__.py``` 

insert this :

```
from flask import Flask

app = Flask(__name__)

# DO NOT move up !!! Above line app = Flask(__name__) = initiates Flask class ! import must be after !
from my_flask_app_package import routes

if __name__ == "__main__":
    app.run(host='0.0.0.0')
  
```

save.

## creates routes.py
``` nano routes.py```

insert this 
```
#from flask import redirect, url_for # here comes more import as you need fro flask
from my_flask_app_package import app


@app.route("/")
def root_():
    return 'this is home page'

```

save

## modify wsgi

``` cd .. ; nano wsgi.py```

MODIFY content to this :
```
from my_flask_app_package import app as application

if __name__ == "__main__":
    application.run()

```

save
## delete run.py
``` rm run.py ```

## restastr server 

visit IP ;)

# and be happy :)


