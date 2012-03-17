---
date: '2012-03-16 23:25:39'
layout: post
slug: multiple-python-apps-with-nginx-uwsgi-emperor-upstart
status: publish
title: Running multiple python apps with nginx and uwsgi in emperor mode
---

{{ page.title }}
================

This is a recipe on how to **easily** run multiple Python web applications using uwsgi server (in emperor mode) and behind nginx.
Most existing docs and blogs would show how to manually start uwsgi to run a single app. In this post, I'll show how to configure uwsgi as a system service (with upstart) capable of serving multiple python WSGI compliant web applications by simply placing them in a standard location and adding an ini file.

It took me many many hours and sleepless nights to get this setup to work. I was on the verge of giving up on uwsgi/nginx and falling back to apache/mod python. I'm not proud of it, but when it worked, I had (and still don't have) no idea why it did. So I would advice the readers not to change too much at a time, and rather work incrementally, with small changes at a time and immediate testing afterwards.

I'm using this setup on a couple EC2 machines running Ubuntu 11.10, nginx 1.0.5-1, Python 2.7.2, uwsgi 0.9.8.1-1 and using the bottle web framework, but the same **should** work for any other WSGI compliant framework.


## 1. Installing the required software

* We'll (of course) need Python. It comes pre-installed on Ubuntu and other linuxes. Nothing to do here.
* uwsgi: `sudo aptitude install uwsgi uwsgi-plugin-python`
* bottle (optional): Since I'll be using Bottle framework for the test applications, we'll need to have it installed for this recipe to work, but if you're using another framework, you could skip this step. Just run this command: `sudo easy_install bottle`. If `easy_install` isn't already installed, run `sudo aptitude install python-setuptools`.
* nginx: `sudo aptitude install nginx-full`

## 2. Configuration
### uwsgi
We'll run uwsgi using upstart so that it is automatically started on startup.

We'll create a `uwsgi.conf` file in `/etc/init`:

`sudo emacs /etc/init/uwsgi.conf`:

With this content:

{% highlight bash %}
# uWSGI - manage uWSGI application server                                                                                                                                                                
#                                                                                                                                                                                                    

description     "uWSGI Emperor"

start on (filesystem and net-device-up IFACE=lo)
stop on runlevel [!2345]

respawn

env LOGTO=/var/log/uwsgi.log
env BINPATH=/usr/bin/uwsgi
env EMPEROR_INI_GLOB=/home/ubuntu/apps/*/*.ini

exec $BINPATH --emperor $EMPEROR_INI_GLOB --logto $LOGTO
{% endhighlight %}

Notice the glob telling uwsgi emperor mode where to look for application's configuration file (I'm using the ini syntax here). In my case, it points to all the ini files in any sub directory of the apps directory in my home dir. **You'll need to modify that to match your setup**.

Then run this command to refresh the services configuration:

`sudo initctl reload-configuration`

We also want to make sure than uwsgi is using the system installed python (2.7 in this case) and not another version (2.6), so we need to run this:

`sudo update-alternatives --set uwsgi /usr/bin/uwsgi_python27`

And start the uwsgi service:

`sudo service uwsgi start`

### nginx
We'll create a `apps.conf` file in the `/etc/nginx/sites-enabled` directory to tell nginx to pass through request to uwsgi:

`sudo emacs /etc/nginx/sites-enabled/apps.conf`

{% highlight nginx %}
server {
  listen 80;
  server_name jawher.me;
  index index.html;

  location /app1/ {
    include uwsgi_params;
    uwsgi_param SCRIPT_NAME /app1;
    uwsgi_modifier1 30;
    uwsgi_pass unix://tmp/app1.sock;
  }

  location /app2/ {
    include uwsgi_params;
    uwsgi_param SCRIPT_NAME /app2;
    uwsgi_modifier1 30;
    uwsgi_pass unix://tmp/app2.sock;
  }
}
{% endhighlight %}

You'll need to customize this file to match your setup (server name, apps locations, socket names, etc.)

Also, there's a bit of magic here, the `uwsgi_param SCRIPT_NAME /app?` and `uwsgi_modifier1 30` thingies. Without them, requesting `jawher.me/app1/index` would end up calling `app1/index` on the bottle web application (i.e. the nginx location gets passed to the wsgi application, which is not what we want most of the time). With this modifier and script param thingies, the nginx location is stripped before invoking the python application.

Start the nginx server if it is not already running:

`sudo service nginx start`

Also, whenever you change the nginx config files, remember to run the following command:

`sudo service nginx reload`

## 3. The test applications
We'll create to test web applications to test our setup:

An app in this case is a python file that defines a bottle app:

`emacs /home/ubuntu/apps/app1/app1.py`

{% highlight python %}
import bottle
import os

@bottle.get('/index')
def index():
    return "App 1 live and kicking"

if __name__ == '__main__':
    bottle.debug(True)
    bottle.run(reloader=True, port=8080)
else:
    os.chdir(os.path.dirname(__file__))
    application = bottle.default_app()
{% endhighlight %}

This file can be executed with python, in which case the main bloc will launch the application in debug mode, or with uwsgi without a change to the file. Don't change the variable name `application`, as that's what uwsgi will be looking for (unless you tell it otherwise).

We'll also need an ini file to tell uwsgi emperor mode how to handle our app:

`emacs /home/ubuntu/apps/app1/app1.ini`

{% highlight ini %}
[uwsgi]
master = true
processes = 1
vacuum = true
chmod-socket = 666
socket = /tmp/%n.sock
uid = www-data
pythonpath = %d
module = app1
{% endhighlight %}

The second application is exactly the same, except for 'app 1' which gets replaced by 'app 2':

`emacs /home/ubuntu/apps/app2/app2.py`

{% highlight python %}
import bottle
import os

@bottle.get('/index')
def index():
    return "App 2 live and kicking"

if __name__ == '__main__':
    bottle.debug(True)
    bottle.run(reloader=True, port=8080)
else:
    os.chdir(os.path.dirname(__file__))
    application = bottle.default_app()
{% endhighlight %}

`emacs /home/ubuntu/apps/app1/app2.ini`

{% highlight ini %}
[uwsgi]
master = true
processes = 1
vacuum = true
chmod-socket = 666
socket = /tmp/%n.sock
uid = www-data
pythonpath = %d
module = app2
{% endhighlight %}

# 4. Troubleshooting
When it doesn't work, you'll usually end up with an unhelpful 502 error. To diagnose the problem:

* Check that the sockets were created: `ls /tmp` and that nginx's process can read, write and execute them. This shouldn't happen though as we specified `chmod 666` in the applications config file.
* Check that the apps were correctly loaded in `/var/log/uwsgi.log`: if you find this message `\*\*\* no app loaded. going in full dynamic mode \*\*\*`, then go back and double check the module name in the app ini file, or try to run the app with python, as in `python app1.py` to see if there aren't any missing imports or other errors.
* Make sure that your app runs with the same version of python that uwsgi uses. Usually, you'll just want to configure uwsgi to use the system-wide python install.
* `/var/log/nginx/error.log` might turn out some useful info too.