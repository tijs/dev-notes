# Setting up a new site on our server

This document describes how to setup a new site on our Ubuntu 11.10 server at Rimuhosting.

## Preparation

Setting up the directory structure and configuration for a new site we'll call _sitename_ for now.

### Create a new Virtualenv

    virtualenv sitename.com
    cd sitename.com
    source bin/activate

### Check out the source

    git clone git@github.com:Automatique/projectname.git
    
### Install all the requirements

The project from the github repo should contain a requirements.txt file containing all the python packages (+versions) needed for this app to run.

Before you install the requirements double check that Gunicorn and psycopg2 are already in the requirements as they will not be available in the virtualenv.

    cd projectname
    pip install -r requirements.txt

### Create the dirs

Create the dirs to hold uploaded media and the server logs.

    cd ..
    mkdir data
    mkdir logs
    chmod 777 data logs
    
### Create a new Postgres database

Before we can run the site we need to create a database for our app. To do this login as postgres and then create a database using a user for the app or the general _django_login_ user.

    su postgres
    psql template1
    
Once logged in create the database like so.

    CREATE DATABASE projectname_db OWNER django_login ENCODING 'UTF8';

Before we can use the database we need to add access permissions to Postgresql.

    su tijs
    sudo vi /etc/postgresql/9.1/main/pg_hba.conf

Add the following line.

    local    projectname_db   django_login   md5

And then reload Postgresql.

    sudo service postgresql reload
    
## Web server configuration

### Running Gunicorn as a service

We are going to create a new directory for our webapp service:

    sudo mkdir /etc/service/projectname
    sudo vi /etc/service/projectname/run

Now create a runscript that will look similar to this. This will start gunicorn_django (and set the proper ENV setting) using the configuration in the project directory.

    #!/bin/sh

    ROOT=/home/tijs/projects/sitename.com
    GUNICORN=/bin/gunicorn_django
    PID=/var/run/projectname.pid

    if [ -f $PID ]
        then rm $PID
    fi

    cd $ROOT
    exec env AUTOMATIQUE=1 $ROOT$GUNICORN -c $ROOT/gunicorn.conf.py --pid=$PID

Make sure the run script is executable. And stop runit from trying to run gunicorn until we are ready

    sudo chmod +x /etc/service/projectname/run
    sudo sv stop projectname

Next step is creating the project configuration for gunicorn.

    vi /home/tijs/projects/sitename.com/gunicorn.conf.py
    
Which should look something like this (remember it's a python file!). Make sure your using a free port number and check that the logs directory is available and writable. The python path is where your app was checked out.

    ROOT = "/home/tijs/projects/sitename.com"

    bind = "127.0.0.1:8003"
    workers = 2
    loglevel = "debug"
    errorlog = "%s/logs/gunicorn.log" % ROOT
    pythonpath = "%s/projectname" % ROOT
    django_settings = "settings"
    
Now test if the startup script actually runs

    sudo /etc/service/projectname/run
    
And if not get a bit more info..

    sudo bash -x /etc/service/projectname/run
    
If so start the runit service

    sudo sv start projectname
    sudo sv status projectname
    

### Setup Nginx

    vi nginx.conf
    
Which should look something like this:

    server {
        listen 80;
        server_name sitename.com;
    
        access_log /home/tijs/projects/sitename.com/logs/nginx.access.log;
        error_log /home/tijs/projects/sitename.com/logs/nginx.error.log;
    
        root /home/tijs/projects/sitename.com/projectname;
    
        location /media/admin/ {
            alias /home/tijs/projects/sitename.com/projectname/static_root/admin/;
        }
    
        location /media/ {
            alias /home/tijs/projects/sitename.com/data/;
            # if asset versioning is used
            if ($query_string) {
                expires max;
            }
        }
    
        location /static/ {
            alias /home/tijs/projects/sitename.com/projectname/static_root/;
            # if asset versioning is used
            if ($query_string) {
                expires max;
            }
        }
    
        location / {
            proxy_pass_header Server;
            proxy_set_header Host $http_host;
            proxy_redirect off;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Scheme $scheme;
            proxy_connect_timeout 10;
            proxy_read_timeout 10;
            proxy_pass http://localhost:8003/;
        }
    }

The _server_name_ can be made up as long as were testing. The port number (8003) should be unique for this site on this server.

To redirect requests to the www subdomains to the naked root add this to the top of the _nginx.conf_.

    server {
        # rewrite www to non-www domain with 301 redirect
        listen 80;
        server_name www.sitename.com;
        rewrite ^ http://sitename.com$request_uri? permanent;
    }
    
Once your config is ready make sure it's picked up by nginx by adding a symlink to the config in sites-enabled.

    sudo ln -s /home/tijs/projects/sitename.com/nginx.conf /etc/nginx/sites-enabled/sitename.com
    
To enable the site check if the config is correct, then reload the Nginx configuration.

    sudo nginx -t
    sudo nginx -s reload
