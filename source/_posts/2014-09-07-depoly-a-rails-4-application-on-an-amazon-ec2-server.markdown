---
layout: post
title: "Depoly a rails 4 application on an Amazon EC2 server"
date: 2014-09-07 17:45:05 +0930
comments: true
categories: rails deploy deployment
---

In this blog I will list the steps needed to deploy a Rails 4 application on an Amazon EC2 server. The OS is Ubuntu Server 14.04 LTS, the database is Postgresql and the web server is Unicorn + Ngnix. Previously I mainly deploy the project on Engine Yard. But this time I need to deploy it on a server manually. 

## Updating And Preparing The Operating System

### Update the Operating System

Firstly we need to update our OS by running following command,

{% codeblock lang:bash %}

sudo apt-get update

{% endcodeblock %}

### Change locale file
In some cases on my server it shows the following warnings,

{% codeblock %}
perl: warning: Setting locale failed.   
perl: warning: Please check that your locale settings:   
        LANGUAGE = "en_US:en",   
        LC_ALL = (unset),   
        LC_MESSAGES = "en_US.UTF-8",   
        LANG = "en_US.UTF-8"   
    are supported and installed on your system.
{% endcodeblock %}

The solution is to edit the **/etc/default/locale** and add the following contents,

{% codeblock %}
LC_ALL=en_US.UTF-8
LANG=en_US.UTF-8
{% endcodeblock %}

### Install the necessary packages

Next we need to install the following necessary packages by running,

{% codeblock lang:bash %}

sudo apt-get install build-essential openssl libreadline6 libreadline6-dev zlib1g zlib1g-dev libssl-dev git-core python-software-properties nodejs libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt-dev autoconf libc6-dev ncurses-dev automake libtool bison pkg-config

{% endcodeblock %}

The *python-software-properties* allows us to easily manage your distribution and independent software vendor software sources. It will be useful when we install Postgresql and Nginx.

The *nodejs* is a Javascript runtime for Rails.

## Install rbenv
Now we need to install a Ruby runtime. I choose rbenv instead of rvm, which is more lightweight for production.

Let's install rbenv by running the following command,

{% codeblock lang:bash %}

curl https://raw.githubusercontent.com/fesplugas/rbenv-installer/master/bin/rbenv-installer | bash

{% endcodeblock %}

After installation, following its instructions, add the following content at the top of ~/.bash_profile

{% codeblock lang:bash %}
export RBENV_ROOT="${HOME}/.rbenv"

if [ -d "${RBENV_ROOT}" ]; then
  export PATH="${RBENV_ROOT}/bin:${PATH}"
  eval "$(rbenv init -)"
fi
{% endcodeblock %}

After that run `source ~/.bash_profile` to make the changed effective.

Now we install Ruby 2.1.2 and make it global by running following commands,

{% codeblock lang:bash %}
rbenv install 2.1.2 --verbose
rbenv rehash
rbenv global 2.1.2
{% endcodeblock %}

The install could take a while so grab a cup of coffee.

After Ruby 2.1.2 is installed, we can install bundler by typing,

{% codeblock lang:bash %}

gem install bundler
rbenv rehash

{% endcodeblock %}

## Install Nginx

Now let's install Nginx. Let's add the repository of Nginx by typing following command,

{% codeblock lang:bash %}

sudo apt-add-repository ppa:nginx/stable

{% endcodeblock %}

Now we need to invoke `sudo apt-get update` again since we added a new software source.

{% codeblock lang:bash %}

sudo apt-get update

{% endcodeblock %}

Now let's install Nginx by invoking,

{% codeblock lang:bash %}
sudo apt-get install nginx
{% endcodeblock %}

Let's check Nginx status by typing

{% codeblock lang:bash %}
sudo service nginx status
{% endcodeblock %}

If it's not running, we can start it by running following command,

{% codeblock lang:bash %}
sudo service nginx start
{% endcodeblock %}

## Install Unicorn

Now we need to install unicorn if you haven't done so.

Add `gem 'unicorn'` in your **Gemfile** and run `bundle install`.

And after that you can run `unicorn_rails` to start it. The default port of unicorn is 8080.


And then create unicorn config file **config/unicorn.rb** which has following contents,

{% codeblock lang:ruby %}
working_directory File.expand_path("../..", __FILE__)
worker_processes 5
listen "/tmp/unicorn.sock"
timeout 30
pid "/tmp/unicorn_xhoppe.pid"
stdout_path "/data/xhoppe/log/unicorn.log"
stderr_path "/data/xhoppe/log/unicorn.log"
{% endcodeblock %}

Here we set worker_processes to 5, and in the server our application will be at **/data/xhoppe**. And the stdout_path and stderr_path are set to **/data/xhoppe/log/unicorn.log**. 

## Install Postgresql

Now let's install Postgresql. 

I want to install Postgresql 9.3 but the Postgresql version in default repository is 9.1. So we install Postgresql 9.3 by typing following commands, [refer here](http://wiki.postgresql.org/wiki/Apt)

{% codeblock lang:bash %}
sudo apt-get install wget ca-certificates
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install postgresql-9.3 pgadmin3 libpq-dev
{% endcodeblock %}

Now let's create a Postgresql role, open psql by running `sudo su -c psql postgres` and type following commands,

{% codeblock %}
create role xhoppe with password 'xhoppe_secret';
alter role xhoppe createdb;
alter role xhoppe WITH LOGIN;
{% endcodeblock %}

## Nginx config file
And then update the **/etc/nginx/sites-enabled/default** file,

{% codeblock %}

upstream unicorn {
 server unix:/tmp/unicorn.sock fail_timeout=0;
}

server {
  listen 80 default_server;

  root /data/xhoppe/public;

  error_page 404 /data/xhoppe/public/404.html;
  error_page 500 502 503 504 /data/xhoppe/public/500.html;

  location / {
    try_files $uri/index.html $uri @unicorn;
  }

  location @unicorn {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://unicorn;
  }
}

{% endcodeblock %}

The upstream unicorn matches the `listen "/tmp/unicorn.sock"` in config/unicorn.rb

After that, we need to run `service nginx restart` to restart nginx.

## Clone project

Now we use git clone to clone our project in **/data/xhoppe**

First we define a **/data/xhoppe/env.sh** which defines some environments,

{% codeblock %}
XHOPPE_DATABASE_PASSWORD=secret
export XHOPPE_DATABASE_PASSWORD

SECRET_KEY_BASE=9a5a0316a...
export SECRET_KEY_BASE

{% endcodeblock %}

The **XHOPPE_DATABASE_PASSWORD** is the production database's password. And the SECRET_KEY_BASE is for rails session, which is generated by running `rake secret`.

After that we can run `source env.sh` to set these environment variables.

## Running the project.

Now we can run the project by running

`unicorn -c config/unicorn.rb -E production -D`

Now we can access the website from our browser.








