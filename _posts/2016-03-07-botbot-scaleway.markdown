---
layout: post
title: "Installing botbot.me on Scaleway"
---

[BotBot.me](http://botbot.me) is a logging IRC bot. [Scaleway](https://scaleway.com) provides bare-metal ARM servers for €3/mo. Can we combine them? We can!

Cribbed from the [real install instructions](https://botbot.readthedocs.org/en/latest/install.html), with extensions as necessary.

1. Instantiate a new server based on the Ubuntu Wily (15.10) image
1. Install botbot.me dependencies: `apt-get install postgresql-contrib-9.4 postgresql-server-dev-9.4 python-dev virtualenv golang-go redis-server git build-essential`
1. Install [node.js](https://github.com/nodesource/distributions): `curl -sL https://deb.nodesource.com/setup_5.x | bash - && apt-get install -y nodejs`
1. Create a botbot user with `adduser botbot --disabled-password`
1. Create a botbot postgres user with `sudo -iu postgres createuser -dP botbot` and assign a secure password (consider `pwgen 24 1`). You _do_ need to configure password authentication because of how botbot interacts with lib/pg.
1. Switch to the botbot user with `su botbot`, change to the home directory with `cd`
1. `virtualenv botbot && source botbot/bin/activate`
1. `pip install -e git+https://github.com/BotBotMe/botbot-web.git#egg=botbot`
1. `cd $VIRTUAL_ENV/src/botbot`
1. `make -j4 dependencies NPM_BIN=/usr/bin/npm`
1. Edit `.env`; set SECRET_KEY and WEB_SECRET_KEY to good random strings (consider `pwgen 24 2`). Set `STORAGE_URL` to `STORAGE_URL='postgres://botbot:YOUR_PSQL_PASSWORD@localhost/botbot'`. Set `WEB_PORT='0.0.0.0:8000'` or set up appropriate SSH forwarding. Comment out `PUSH_STREAM_URL`.
1. Export .env with `export $(cat .env | grep -v ^# | xargs)`
1. `createdb botbot`
1. As root: `echo "create extension hstore" | sudo -iu postgres psql botbot`
1. As botbot again: `manage.py migrate`
1. `manage.py createsuperuser`
1. `honcho start`

It's alive! Proceed to the [setup instructions](https://botbot.readthedocs.org/en/latest/getting_started.html). Note that your bot will not connect to Freenode until you reload the configuration in "add a channel" step 7.

To use a real web server, set up nginx and uwsgi. Install uwsgi into your virtualenv and `apt-get install nginx`. It will fail; this is fine. Edit `/etc/nginx/sites-available/default` and comment out `listen [::]:80 default_server;`. Run `apt-get install -f` to make dpkg happy.

Add to sites-enabled:

{%highlight nginx%}
upstream botbot {
	server localhost:8000;
}

server {
	listen 80;
	server_name your_website.com;
	charset utf-8;
	
	client_max_body_size 100M;
	
	location /static {
		alias /home/botbot/botbot/var/static;
	}
	
	location / {
		uwsgi_pass botbot;
		include /etc/nginx/uwsgi_params;
	}
}
{%endhighlight%}

Edit the Procfile to launch the web job like `uwsgi --socket localhost:8000 --module botbot.wsgi`. Run `manage.py collectstatic` to collect the static files. Run `honcho start` again and you should be in business!

Ideally, this is where you write systemd configuration files to launch the bot at startup... or you could run it in `tmux` like me. `#devops` Someone tell me how to do this and I'll thank you.
