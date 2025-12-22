---
layout: post
title: "Oveleaf On-Premise"
date: 2025-12-22 10:00:00 -0600
categories: blog
---

## The Problem
I’ve been considering deploying a company-wide Overleaf instance for some time. 
We didn’t want to rely on yet another managed service, and it seemed like a straightforward process.

While I found several online guides, most had small differences or were missing details 
that required extra tinkering with the settings. I decided to write this guide to 
help our company admins maintain the service in the future and to assist anyone else who 
might need it.

What follows describes how to set up the Overleaf Community Edition, configure TLS, DNS, and SMTP settings, and perform basic administrative tasks such as listing and removing users and managing backups.

## Prerequisites
* Fresh ubuntu 24 LTS server with at least 2 CPU cores and 4GB memory
* Add 1 CPU core and 1 GB of memory for every **five** more concurrent users.
* (DNS) If you are hosting **example.com** and Overleaf will live in 1.2.3.4, seetup a record that points **example.com** and **www.example.com** to 
the IPv4 address 1.2.3.4
* You may want to harden the system, just remember that overleafs compiles are not isolated in community edition.
* Install [Docker](https://docs.docker.com/engine/install/ubuntu/)
* Install [Overleaf](https://github.com/overleaf/toolkit) following this [guide](https://github.com/overleaf/toolkit/blob/master/doc/quick-start-guide.md)
* (TLS) prep the config files if you will use TLS with: bin/init --tls

## Configuration
Note that when you change configurations you must do:
```
sudo ./bin/stop
sudo ./bin/up -d
```
Or the container will not be updated.

### Customize

* In /opt/overleaf/config/variables.env uncomment (remove #) and customize

```
SHARELATEX_APP_NAME=Our Overleaf Instance
# SHARELATEX_SITE_URL=http://overleaf.example.com
# SHARELATEX_NAV_TITLE=Our Overleaf Instance
# SHARELATEX_HEADER_IMAGE_URL=http://somewhere.com/mylogo.png
# SHARELATEX_ADMIN_EMAIL=support@example.com

# SHARELATEX_LEFT_FOOTER=[{"text":"Powered by Overleaf © 2021", "url": "https://www.overleaf.com"}, {"text": "Contact your support team", "url": "mailto:support@example.com"} ]
# SHARELATEX_RIGHT_FOOTER=[{"text":"Hello I am on the Right"}]
```

For example:

```
SHARELATEX_APP_NAME=Overleaf

SHARELATEX_SITE_URL=https://www.example.com
SHARELATEX_NAV_TITLE=EXAMPLE.COM
SHARELATEX_HEADER_IMAGE_URL=https://www.example.com/mylogo.png
SHARELATEX_ADMIN_EMAIL=admin@example.com

SHARELATEX_LEFT_FOOTER=[{"text":"Powered by Overleaf", "url": "https://www.overleaf.com"}, {"text": "Contact us", "url": "mailto:admin@example.com"} ]
SHARELATEX_RIGHT_FOOTER=[{"text":"©2022 EXAMPLE.COM. All rights reserved."}]
```

### Nginx
The toolkit uses an internal Nginx,
* In /opt/overleaf/config/overleaf.rc

```
NGINX_ENABLED=true
NGINX_HTTP_LISTEN_IP=<server-instance-external-ip>
NGINX_TLS_LISTEN_IP=<server-instance-external-ip>
```

### TLS
* In /opt/overleaf/config/variables.env uncomment (remove #)
```
SHARELATEX_BEHIND_PROXY=true
SHARELATEX_SECURE_COOKIE=true
```

## Update Latex

```
bin/shell
tlmgr update --self
tlmgr option repository https://mirrors.mit.edu/CTAN/systems/texlive/tlnet/
nohup tlmgr install scheme-full &

```

* Check state with tail nohup.out 
* When you see the following, it's done!
```
....
tlmgr: package log updated: /usr/local/texlive/2021/texmf-var/web2c/tlmgr.log
tlmgr: command log updated: /usr/local/texlive/2021/texmf-var/web2c/tlmgr-commands.log
```

* You can now upgrade those packages:
```
tlmgr update --self --all
exit
```

* Now we need to save and update the container
```
docker commit sharelatex sharelatex/sharelatex:with-texlive-full
```
* Setup an override for the image in overleaf/lib/docker-compose.override.yml
```
---
services:
  sharelatex:
    image: sharelatex/sharelatex:with-texlive-full
```
```
bin/stop && bin/docker-compose rm -f sharelatex && bin/up -d
```

I had problems with this step so I ended up doing it manually:
```
sudo ./bin/docker-compose -f lib/docker-compose.base.yml -f lib/docker-compose.override.yml up -d --no-build --pull never
```

Verify that the new container is actually being used:
```
sudo docker ps --format "{{.Names}}: {{.Image}}"
```

Should show:

```
sharelatex: sharelatex/sharelatex:with-texlive-full <<<---- If you don't see with-textlive-full you must thinker until you fix this step otherwise the wrong container is being used
redis: redis:7.4
nginx: nginx:1.28-alpine
mongo: mongo:8.0
```

Now you can start the server again and all the packages should be ready!

## SMPT
The following is specific for gmail, you may have to thinker a bit with the option for your mail server.
```
# config/variables.env

OVERLEAF_EMAIL_SMTP_HOST=smtp.gmail.com
OVERLEAF_EMAIL_SMTP_PORT=587
OVERLEAF_EMAIL_SMTP_SECURE=false
OVERLEAF_EMAIL_SMTP_USER=your_user@gsuite.example.us
OVERLEAF_EMAIL_SMTP_PASS=--2SV PP Password, will not work with basic credentials
OVERLEAF_EMAIL_SMTP_TLS_REJECT_UNAUTH=true
OVERLEAF_EMAIL_SMTP_IGNORE_TLS=false
OVERLEAF_EMAIL_SMTP_LOGGER=true <----- This is useful to debug what is going wrong
```

Once you send an invite from the admin console, you can check if it was successful using:
```
sudo ./bin/logs web
```

## Admin
### First Login
http://www.example.com/lauchpad and register admin user

### List users
```
# get the container ID using: docker ps
docker exec <mongo-container-id> sh -c 'exec mongoexport -d sharelatex -c users -f email --type=csv' > sharelatex-emails.csv
```

#### Delete User
```
bin/docker-compose exec sharelatex /bin/bash -ce "cd /overleaf/services/web && node modules/server-ce-scripts/scripts/delete-user.mjs --email=<email-to-delete>"
```

#### Reset User Password
This gets a little complicate. If you set SMPT users should be able to reset their passwords through the UI.

* First, we need to retrieve a token for the user using its email (in the example test@example.org):

```
sudo bin/docker-compose exec sharelatex /bin/bash -ce "cd /overleaf/services/web && node modules/server-ce-scripts/scripts/create-user --email=test@example.org"
...
Please visit the following URL to set a password for test@example.org and log in:

  https://www.example.com/user/activate?token=<token>&user_id=<user-id>

```

We can then give the user the following link to reset their password:

```
https://www.example.com/user/password/set?passwordResetToken=<token>&user_id=<user-id>
```

### Backup
Backing up Overleaf data essentially boils down to backing up three directories:

* ~/sharelatex_data
* ~/mongo_data
* ~/redis_databackup-new/

```
# backup_save.sh

# Gracefully shutdown the old instance
bin/stop

# Create the tar-ball
tar --create --file backup-old-server.tar config/ data/

# Copy the backup-old-server.tar file from the old-server to the new server
```

```
# backup_load.sh

# Gracefully shutdown new instance (if started yet)
bin/stop

# Move new data, you can delete it too
mkdir backup-new-server
mv config/ data/ backup-new-server/

# Populate config/data dir again
tar --extract --file backup-old-server.tar

# Start containers
bin/up
```
