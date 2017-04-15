This tutorial is for manually setting up a production instance of Mastodon on a fresh Ubuntu server. 

# Part A

Complete part A as root.

## upgrade your server

	apt-get update
	apt-get upgrade

## install dependencies

	apt-get install imagemagick ffmpeg libpq-dev libxml2-dev libxslt1-dev file git curl

	curl -sL https://deb.nodesource.com/setup_4.x | sudo bash -
	apt-get install nodejs

	npm install -g yarn

	apt-get install redis-server redis-tools
	apt-get install postgresql postgresql-contrib

## Set up Postgres

	su - postgres
	psql
	CREATE USER mastodon CREATEDB;
	\q

### Postgres on Ubuntu 16.04

You only need to complete these instructions for Ubuntu 16.06

You need to explicitly enable ident authentication so that local users can connect to the database without a password:

	sed -i '/^local.*postgres.*peer$/a host    all     all     127.0.0.1/32    ident' /etc/postgresql/9.?/main/pg_hba.conf

Then install an ident daemon, which does not come installed by default:
	
	apt-get install pidentd
	systemctl enable pidentd
	systemctl start pidentd
	systemctl restart postgresql

## nginx

It's good to get this started before hand so the certbot works

	sudo apt-get install nginx

Create a really simple server configuration file
	
	server {
	        listen 80;
	        server_name domain.tld;
	        root /home/mastodon/live/public;
	        index index.html;
	}

Then run certbot to generate the certificate for the domain.

	sudo certbot --nginx -d domain.tld

Now you have working nginx server with an SSL certificate.

## rbenv

Install [rbenv](https://github.com/rbenv/rbenv#installation) and [rbenv-build](https://github.com/rbenv/ruby-build#installation)

## mastodon user

Create a 'mastodon' user on the server to run the application.

	useradd -m -d /home/mastodon mastodon

Set the password for the mastodon account

	passwd mastodon

# Part B

## Switch accounts

Switch to the mastodon account to complete this section of instructions

	sudo su - mastodon 

## git

Move to the mastodon home directory and clone from github

	cd ~
	git clone https://github.com/tootsuite/mastodon.git live
	cd live

Install ruby 2.4.1 -- this may take some time

	rbenv install 2.4.1

Now we can get gems like bundler

	gem install bundler
	bundle install --deployment --without development test
	yarn install

## Configuration

Copy the sample environment file

	cp .env.production.sample .env.production

In another terminal window generate 3 random strings using

	rake secret

You'll need those in the .env.production file

Open the environment file in the text editor of your choice. Nano is used here simply because it's easy to use. 

	nano .env.production

Fill out the settings in the environments files.

The values for Redis, unless you're doing something custom, are

	REDIS_HOST=localhost
	REDIS_PORT=6379

The values for the database, unless you're doing something custom, are

	DB_HOST=/var/run/postgresql
	DB_USER=mastodon
	DB_NAME=mastodon_production
	DB_PASS=
	DB_PORT=5432

Federation settings are you instance domain and whether you have SSL. If you're following this tutorial, you will have it from certbot.

	LOCAL_DOMAIN=domain.tld
	LOCAL_HTTPS=true

For SMTP settings, you can go as simple as local mail setting to something that will scale up better like Mailgun.

## Rails Setup

Setup the database for the first time, this will create the tables and basic data:

	RAILS_ENV=production bundle exec rails db:setup

Pre-compile all CSS and JavaScript files:

	RAILS_ENV=production bundle exec rails assets:precompile

## Cronjobs

There are several tasks that should be run once a day to ensure that mastodon is running smoothly. 

Add the following to crontab by running `crontab -e` and adding the following

```
RAILS_ENV=production
@daily cd /home/mastodon/live && /home/mastodon/.rbenv/shims/bundle exec rake mastodon:media:clear > /dev/null
@daily cd /home/mastodon/live && /home/mastodon/.rbenv/shims/bundle exec rake mastodon:push:refresh > /dev/null
@daily cd /home/mastodon/live && /home/mastodon/.rbenv/shims/bundle exec rake mastodon:feeds:clear > /dev/null
```

----

# Part C

Exit out of the mastodon account to switch back to root.

Complete Part C as root.

## Systemd

### Web workers

Create the file `/etc/systemd/system/mastodon-web.service` and place the following in it:

	systemd
	[Unit]
	Description=mastodon-web
	After=network.target
	
	[Service]
	Type=simple
	User=mastodon
	WorkingDirectory=/home/mastodon/live
	Environment="RAILS_ENV=production"
	Environment="PORT=3000"
	ExecStart=/home/mastodon/.rbenv/shims/bundle exec puma -C config/puma.rb
	TimeoutSec=15
	Restart=always
	
	[Install]
	WantedBy=multi-user.target

### Background workers

Create the file `/etc/systemd/system/mastodon-sidekiq.service` and place the following in it:

	systemd
	[Unit]
	Description=mastodon-sidekiq
	After=network.target
	
	[Service]
	Type=simple
	User=mastodon
	WorkingDirectory=/home/mastodon/live
	Environment="RAILS_ENV=production"
	Environment="DB_POOL=5"
	ExecStart=/home/mastodon/.rbenv/shims/bundle exec sidekiq -c 5 -q default -q mailers -q pull -q push
	TimeoutSec=15
	Restart=always
	
	[Install]
	WantedBy=multi-user.target

### Streaming API

Create the file `/etc/systemd/system/mastodon-streaming.service` and place the following in it:

	systemd
	[Unit]
	Description=mastodon-streaming
	After=network.target
	
	[Service]
	Type=simple
	User=mastodon
	WorkingDirectory=/home/mastodon/live
	Environment="NODE_ENV=production"
	Environment="PORT=4000"
	ExecStart=/usr/bin/npm run start
	TimeoutSec=15
	Restart=always
	
	[Install]
	WantedBy=multi-user.target

To activate these, run

	sudo systemctl enable /etc/systemd/system/mastodon-*.service
	sudo systemctl start mastodon-web.service mastodon-sidekiq.service mastodon-streaming.service


## Things to look out for when upgrading Mastodon

You can upgrade Mastodon with a `git pull` from the repository directory. You may need to run:

- `RAILS_ENV=production bundle exec rails db:migrate`
- `RAILS_ENV=production bundle exec rails assets:precompile`

Depending on which files changed, e.g. if anything in the `/db/` or `/app/assets` directory changed, respectively. Also, Mastodon runs in memory, so you need to restart it before you see any changes. If you're using systemd, that would be:

    sudo systemctl restart mastodon-*.service
