---
layout: post
title: "Configure Customized data directory for Postgresql on Ubuntu 14.04"
date: 2015-02-07 20:01:19 +1030
comments: true
categories: 
---

**PostgreSQL** is a very popular database in Rails world. Here I describe how to install and configure it on Ubuntu 14.04 server.

## Install
Using apt-get to install PostgreSQL is very simple. Just execute the following two commands,

{% codeblock lang:bash %}

sudo apt-get update
sudo apt-get install postgresql postgresql-contrib

{% endcodeblock %}

Here on my server it will install PostgreSQL 9.3.

During the installation a user named 'postgres' will be created automatically and this user is used to start postgresql.

So we should change the password of user 'postgres'.

{% codeblock lang:bash %}

$ sudo passwd postgres
Enter new UNIX password: 
Retype new UNIX password: 
passwd: password updated successfully

{% endcodeblock %}

The installation will put configuration files in **/etc/postgresql/9.3/main** by default.

{% codeblock lang:bash %}

vagrant@vagrant-ubuntu-trusty-64:/etc/postgresql/9.3/main$ ls -la
total 56
drwxr-xr-x 2 postgres postgres  4096 Feb  7 09:26 .
drwxr-xr-x 3 postgres postgres  4096 Feb  7 09:26 ..
-rw-r--r-- 1 postgres postgres   315 Feb  7 09:26 environment
-rw-r--r-- 1 postgres postgres   143 Feb  7 09:26 pg_ctl.conf
-rw-r----- 1 postgres postgres  4649 Feb  7 09:26 pg_hba.conf
-rw-r----- 1 postgres postgres  1636 Feb  7 09:26 pg_ident.conf
-rw-r--r-- 1 postgres postgres 20687 Feb  7 09:26 postgresql.conf
-rw-r--r-- 1 postgres postgres   378 Feb  7 09:26 start.conf

{% endcodeblock %}

## Configure new data directory

Sometimes to improve performance, we want put postgres data files in its own disk. For example, we may mount a disk in folder **/database** and we want to put all postgres data file there. So let's do that.

Suppose the **/database** folder already exists. We need to change its owner to postgres firstly.

{% codeblock lang:bash %}

sudo chown -R postgres:postgres /database

{% endcodeblock %}

Now we need to initialize this folder as a data folder

{% codeblock lang:bash %}

$ su postgres
$ /usr/lib/postgresql/9.3/bin/initdb -D /database

{% endcodeblock %}

We must su as postgres first since this user also owns the server process.

Now let's stop server by running **sudo service postgresql stop**.

{% codeblock lang:bash %}

sudo service postgresql stop

{% endcodeblock %}

Now we need to update the file **/etc/postgresql/9.3/main/postgresql.conf**, 

change **data_directory = '/var/lib/postgresql/9.3/main'** to **data_directory = '/database'**

PS: You need to stop the server first before updating the data_directory location. Otherwise the stop will fail and you need to kill the process manually.

After update the configuration file we need to restart postgresql server.

{% codeblock lang:bash %}

sudo service postgresql start

{% endcodeblock %}

