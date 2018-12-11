While there are installation instructions in the repositories, I found them lacking as they assume there 
is already a configured system. These instructions are done on a freshly installed Arch based server (Manjaro),
and detail all the steps that I required to get Konishi running.

# Arch systems (Manjaro)
## Zimmerman
### Introduction
I opted to go for a simple sqlite installation, isntead of postgres, so I added the steps for creating and 
configuring the sqlite db.

Also, I couldn't get the register user part of the site working, so one of the steps here is to inject 
a user into the empty database.

### Install prerequisites, clone zimmerman, create venv, install pip reqs
1. `su`
2. `pacman -Syy`
3. `pacman -S python-pip`
4. `mkdir /var/www`
5. `cd /var/www/`
6. `git clone https://github.com/konishi-project/zimmerman.git`
7. `cd zimmerman`
8. `pip install virtualenv`
9. `virtualenv konishienv`
10. `source konishienv/bin/activate`
11. `pip install -r requirements.txt`
12. `chown dorus: /var/www -R`
13. exit root user

### Prep database
1. `cd /var/www/zimmerman`
2. `sqlite3 konishi.db`
3. >> `.exit`
4. In config.py, set: SQLALCHEMY_DATABASE_URI = 'sqlite:////var/www/zimmerman/konishi.db'
sqlite

### Load empty database
1. `source konishienv/bin/activate`
2. `python3`
```
from app import db # Import the database object from app.py
db.create_all() # Create the database cluster
quit() # Exit out of python
```

### Add admin user
```
INSERT INTO user (email,username,first_name,last_name,password,joined_date)
VALUES ('ur@mail.lol','admin','mr','badmin','<<HASH>>',DATETIME());  # I forget how to generate the hash, it's somewhere in the source code, look for the register user function.
```

### Start backend
1. `cd /var/www/zimmerman`
2. `source konishienv/bin/activate`  # if not stil in the venv
3. `python3 app.py`

## Higala
### Install prerequisites && clone
1. `sudo pacman -S npm`
2. `cd /var/www`
3. `git clone https://github.com/konishi-project/higala.git`

### Set config
1. `cd /var/www/higala/src`
2. `nano store.js`
3. Change the backend url from 127.0.0.1:4000 to <your server ip>:4000

### Run 
1. `cd /var/www/higala`
2. `npm install`   # first time only
3. `npm audit fix`  # first time only
3. `npm run serve --fix`   # --fix is first time only

## Running on server via SSH
After running the frontend, do:
1. `ctrl+z` (close process)
2. `bg` (runs last run process in background)
3. `disown` (unlinks it from your account?)

Do the same for the backend, then you can close the terminal and it will keep running.
