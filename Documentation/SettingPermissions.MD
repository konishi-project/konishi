# Permissions

In this document I'll try to explain how the permission structure is set up and why.
This includes creating groups, setting folder/file permissions, and adding users to said groups.

> At the moment of writing a lot of the deployment scripts still use the adhoc permission settings, so don't be 
confused when you read them after reading this. This will soon be updated.

Setting up your own dev environment doesn't require much with regards to setting permissions. I've had it run with default permissions all this time, and it works without a problem. But more than just having it running, we want it secure, and that's what this article is about. 

## Kind of permissions we want to segmentate
Before thinking of what groups to create, it's good to have in view what kind of data we have, and which applications
need to access which. The resources at the time of writing are these:

* SSL certificate
* SSL private key
* Application data (nginx, frontend, backend)
* Database
* User uploaded images

The applications & users are these:

* root
* admin users that log into the vm
* (service) nginx
* (app) backend (zimmerman)
* (app) frontend (higala)

## Running the apps as services
* Nginx will be installed as running as www-data:www-data, so that's the user we'll use to not complicate things too much
* Zimmerman will have to be a persistent service as well, so we'll create a user account for this app later on


## Permissions required per app
Now that we have an overview of all the securables, we need to establish what rights each user will need to have on them, so 
that we can start setting up minimal permissions on all of them.

### Backend (Zimmerman)
* Read + execute rights in `/var/www/zimmerman`
* Write rights in in `/var/www/zimmerman/static/postimages`
* Write rights on `var/www/zimmerman/konishi.db`

### Frontend (Higala)
The frontend is served by nginx, and thus the running website needs no specific permissions
While in dev mode, higala is run through npm as the `ssh user`

### Nginx
* Read + execute rights on `/var/www/konishi.club` (frontend)
* Read rights in `/etc/ssl/certs`
* Read rights in `/etc/ssl/private`
* Standard rights in `/etc/nginx`, and any other auto installed folders

#### Note on flask adhoc SSL [optional reading ;) ]
While Zimmerman is configured to use SSL as well, all the SSL traffic is proxied by nginx. This means Zimmerman does not need access to the CloudFlare cert/key itself, and instead can use an "on the fly" SSL solution. 

Here, flask generates its own SSL certificate during runtime (using pyopenssl) and communicates to nginx via that context.
To achieve this, the following is set in app.py: `app.run(<...>,ssl_context='adhoc')`.

This would normally result in a warning for the user (and indeed, if you'd go to the backend using the VM's IP, you get that warning), but because nginx proxies the connection, all communication to the user uses Cloudflare's certificate. 

I'm not sure on how Cloudflare would fare with this ad hoc solution directly, as the user actually communicates to CloudFlare using their own certificate, and CloudFlare then communicates to the Origin Server (i.e. our VM) via the certificate that we have on our VM. So in any case, the user should never get an error about possibly insecure connections, but CloudFlare might bug out if the cert we use is not the one we set in the CloudFlare settings.

So if all the connection between Nginx and Flask occur behind the screen anyways, why not just use HTTP? In short, I could not get it to work. It should be possible to have flask running in full http, but I keep getting
"No API definition provided." when I try that. Might be some setting in Nginx that I've overlooked.


## Securing the certificate and private key
The standard setting for `/etc/ssl/certs` is `drwxr-xr-x  root root`, so anyone can read and execute files in that folder. Only root can edit. This is perfectly fine for us, as certificates are the things you want to share with the whole wide world anyways.

It's a different question regarding `etc/ssl/private`, where the permissions are standard `drwx------  root root`. So only root can do anything in that folder. I found that out when creating the key with `sudo nano private/keyname.key` and subsequently not being able to `cd` into that directory (sudo cd apparently doesn't work either!). 

To enable easy access for Nginx, and any app that will need it in the future, we'll create the __ssl-cert__ group, set the group of the folder to `ssl-cert`, and set the `/etc/ssl/private` permissions to `640` (owner=read+write, group=read, public=none). As a last step we add `root` and `www-data` to the group `ssl-cert`.

```bash
# Find the user nginx is running as.
# In my case, this is "www-data"
grep user /etc/nginx/nginx.conf

# Create group that will get permissions on /etc/ssl/private
sudo groupadd ssl-cert 

# Add root and www-data to ssl-cert group
sudo usermod -a -G ssl-cert www-data
sudo usermod -a -G ssl-cert root

# Set group of /etc/ssl/private to ssl-cert
sudo chgrp ssl-cert /etc/ssl/private

# Change permissions on /etc/ssl/private
# 640: owner=read+write, group=read, public=none
sudo chmod 640 /etc/ssl/private
```

## Settings permissions for Zimmerman
As half-noted before, Zimmerman is a Flask app. The method of running this until now was as follows:
```bash
# Go to the right directory
cd /var/www/zimmerman

# Open the virtualenv where we installed all our packages
source konishienv/bin/activate

# Run app.py in decoupled mode (the &)
python3 app.py &
```

In this case, it's clear that we're running Zimmerman with our own (`ssh-user`) permissions. Because we want to set this type of user to be able to do a lot more than Zimmerman needs to, it would be better to give Zimmerman it's own user account. This all so that if, god forbids, someone manages to hack our app, they won't be able to do anything of consequence (except for deleting all the uploaded images - which we'll have to setup an external sync for anyways). 

Might as well combine the creating of a new user with setting up a service, so we can easily run (and restart) this app as the correct user.

### Add user

```bash
sudo adduser backend
```
This will automatically create the group `backend` as well. This is important, because we'll need to have r/w/e permissions on `/var/www/zimmerman`, but the user `backend` has to be limited to being able to write only files in `./static/postimages.

We'll set the owner of the folder to `ssh-user` (the account you're currently logged in as), and the group to `backend`. Then we can set restrictive permissions on the `backend` group, and still be able to use git, npm, nano without any troubles using our own account.

```bash
# Set user and group of /var/www/zimmerman to ubuntu:backend
sudo chown -R ubuntu:backend /var/www/zimmerman
```
> Note that I used 'ubuntu' as user name, in your case this might be different name

Enter whatever you want for personal information/password, we should not need that for the service, but write down the password for testing purposes just in case.

### (Finally) set all the permissions
If we look at the permissions on a random file now, we see `-rwxrwxr-- 1 ubuntu backend`. Let's try setting the following settings to all files, and then limit particular files and folders later:
* Owner = all
* Group = all
* Public = none

Special permissions:
* App data  
  * *.py = r/e
  * konishienv = r/e
* Database
  * *.db = r/w/e
* Cache
  * `__pycache__` = r/w/e
* User uploads
  * ./static/postimages/* = r/w/e
  * ./static/postimages = r/e

```bash
# Default: public can do nothing, backend can do everything
sudo chmod -R 770 /var/www/zimmerman

# Set folder and contents to 777: public is allowed to upload files
sudo chmod -R 777 /var/www/zimmerman/static/postimages

# Set folder itself to r+e: public cannot change folder itself
sudo chmod    775 /var/www/zimmerman/static/postimages

# Compiled python file stash, the app should be able to do anything here
sudo chmod -R 770 /var/www/zimmerman/__pycache__

# Be able to read and execute app data as backend, but not edit it
sudo chmod -R 750 /var/www/zimmerman/konishienv
sudo chmod    750 /var/www/zimmerman/*.py

# App should be able to modify, read, and execute the database, of course
sudo chmod    770 /var/www/zimmerman/*.db
```
> I left important permissions in comments even though they are the same as the default, because the default might change.

### Create service for the backend
Instead of using the script shown before to start up Zimmerman, we'll be using a systemd service. I first tried feeding that service the script of before, but once app.py is called, the script is finished and it takes the service with it. We'll have to find another way to initialize the virtualenv then. Luckily the answer was quickly found to be: just call the python executable in the venv. As it is baked into the venv, it will instantiate the venv when called.

This makes the 'executable' part: `/var/www/zimmerman/konishienv/bin/python3 /var/www/zimmerman/app.py`.

Now we can create a service in `/etc/systemd/system`. 
Really all you'd need is the following to start the service as `backend`:
```bash
[Service]
ExecStart=/var/www/zimmerman/konishienv/bin/python3 /var/www/zimmerman/app.py
WorkingDirectory=/var/www/zimmerman
User=backend
Group=backend
```
Easy enough, but while we're at it, let's add some extra configuration. Create a file called `backend.service`: `sudo nano /etc/systemd/system/backend.service`, and paste in the code below.

```bash
[Unit]
Description=Konishi Backend Application

# Make sure the network is initialized first
After=network.target

[Service]
# Runn app.py on start of service
ExecStart=/var/www/zimmerman/konishienv/bin/python3 /var/www/zimmerman/app.py

# Restart Backend when an error causes it to shutdown
Restart=on-failure

# Allow the app to start up for 20 seconds
TimeoutStartSec=20

WorkingDirectory=/var/www/zimmerman

User=backend
Group=backend

[Install]
# Make sure this service is started up after reboot
WantedBy=multi-user.target
```

Then, we can install the service: `sudo systemctl enable backend`. And check whether it's running: `sudo systemctl status backend`. If you change anything to the service file, be sure to reload the daemon:`sudo systemctl daemon-reload` before trying to start it again with `sudo systemctl start backend`.

And just like that we're done! (Hopefully, if you encounter any errors, do let me know `dwrolvink`)

## Setting Nginx permission
Nginx magically has permission to `/var/www/konishi.club`, so I'm gonna have to google that. To be continued.
