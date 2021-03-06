---
layout: post
title: Installing Nginx, UWSGI and Python on Google Compute Engine
category: Google Cloud Platform
tags: [Google Compute Engine]
---

Today I'm gonna show you how to install UWSGI, Python and Nginx on Google Compute Engine instance.

<!--more-->

# Preparation

1.  Download Google Cloud SDK [here](https://cloud.google.com/sdk/downloads)
2.  Create a new project from Google Developers Console - [https://console.developers.google.com](https://console.developers.google.com)
3.  Enable billing for the new project to be able to create Compute Engine instances (just go to the project and click on Compute Engine menu item and you will be prompted to enable billing )
4.  Grant access to your account to the Cloud SDK by logging in. Open a terminal and type in: `$ gcloud auth login` Special confirmation page will be opened in your browser, don't miss it. Confirm the access and return to the terminal. There you will be prompted to enter a project ID you will work with - enter the ID of the project you created before.

# Creating an instance

You can create a new Compute Engine instance with two ways - via web-interface or via command line of the SDK. To create an instance via web-interface open Compute Engine section of the Developers Console and click "Create Instance" button. Let's create a new micro instance with new disk from Debian image (screenshot updated): [![Nginx UWSGI Python on Google Compute Engine](http://lh4.ggpht.com/EUIv7hiyyem90SnwSofAfRoldEkcecxjQP7XbjYYXct8NVxsPNAAjXBGEYQPQ3L3s1i9-EEaAfiGbjttB2qi4Ahy=s600)](http://denisigosite.appspot.com.storage.googleapis.com/Nginx-UWSGI-Python-on-Google-Compute-Engine.png)   Or you can use following command from command line:

``` shell
$ gcutil addinstance "test42" --zone="us-central1-a" --machine_type="f1-micro" --image="https://www.googleapis.com/compute/v1/projects/debian-cloud/global/images/debian-7-wheezy-v20140408"
```

Let's check if instance was created:

``` shell
$ gcutil listinstances
```

[![Screenshot from 2014-04-14 09:53:57](http://lh3.ggpht.com/tpo0cNcZVzoiIa4x8l5fCS_vznUJakhS9KZxVm5e7R7usgrTP27wxlAOaDaR5ZBqRWZ1DUV6iRFA9qoj9S7l31w=s600)](http://denisigosite.appspot.com.storage.googleapis.com/Screenshot-from-2014-04-14-095357.png)

Ok, we can move forward. Lets's SSH to the instance:

``` shell
$ gcutil ssh test42
```

[![Screenshot from 2014-04-14 12:01:33](http://lh3.ggpht.com/7Azr_r38ECnyk2h-SktmuoD3AAxNG_8sD6-6SdmwUJu-u9cKZnZuJxNuDBLcyPRnl0svo3DrRsIYDruHv7tyOpfc=s600)](http://denisigosite.appspot.com.storage.googleapis.com/Screenshot-from-2014-04-14-120133.png) 

Great! We can now install software. But... we still need to add firewall to enable HTTP connections to our instance:

``` shell
$ gcutil addfirewall http --allowed="tcp:http"
```

# Installing software

``` shell
$ sudo apt-get install python-pip
```

Install UWSGI and UWSGI plugin for Python.

``` shell
$ sudo apt-get install uwsgi uwsgi-plugin-python
```

Install Nginx:

``` shell
$ sudo apt-get install nginx
```

# Create our application

Create directory for the application and file with source code:

``` shell
$ sudo mkdir /srv/www/test
$ sudo nano /srv/www/test/app.py
```

Place there simple helloworld app:

``` python
def application(env, start_response):
    start_response('200 OK', [('Content-Type','text/html')])
    return ["Hello World!"]
```

# Configuring and starting application

Create UWSGI configuration file for our application ([see UWSGI docs here](http://uwsgi-docs.readthedocs.org/en/latest/WSGIquickstart.html)):

``` shell
$ sudo nano /etc/uwsgi/apps-available/test.ini
```

Place there this text:

``` ini
[uwsgi]
plugins = python
chdir = /srv/www/test/
wsgi-file = /srv/www/test/app.py
```

Enable this app by creating symbolic link to the apps-enabled dir:

``` shell
$ sudo ln -s /etc/uwsgi/apps-available/test.ini /etc/uwsgi/apps-enabled/
```

Restart UWSGI daemon:

``` shell
$ sudo service uwsgi restart
```

Create Nginx site config file ([see UWSGI docs here](http://uwsgi-docs.readthedocs.org/en/latest/Nginx.html)):

``` shell
$ sudo nano /etc/nginx/sites-available/test
```

``` nginx
server {
    listen 80 default_server;
    location /test {
        include uwsgi_params;
        uwsgi_pass unix:/var/run/uwsgi/app/test/socket;
        uwsgi_param SCRIPT_NAME /;
        uwsgi_modifier1 30;
    }
}
```

Now enable this site by creating a symlink:

``` shell
$ sudo ln -s /etc/nginx/sites-available/test /etc/nginx/sites-enabled/
```

And restart nginx:

``` shell
$ sudo service nginx restart
```

It's time to test it! 

[![Screenshot from 2014-04-14 17:54:51](http://lh6.ggpht.com/gkxZNJVNOK7zjDwIsYLSlh3oJFTCkDB4lNnbFyLcg6ND7swxHziGHv658ijPJgDMrPSP5laCxButIm6rm3nb9Ws=s600)](http://denisigosite.appspot.com.storage.googleapis.com/Screenshot-from-2014-04-14-175451.png)
