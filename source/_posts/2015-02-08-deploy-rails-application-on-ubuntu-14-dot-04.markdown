---
layout: post
title: "Deploy Rails application on Ubuntu 14.04 Part 1"
date: 2015-02-08 22:03:52 +1030
comments: true
categories: postgresql deploy unicorn capistrano
---

In [one of my previous post](http://climber2002.github.io/blog/2014/09/07/depoly-a-rails-4-application-on-an-amazon-ec2-server/) I described how to do the deployment manually on an Amazon EC2 server. And in this new series of posts I want to introduce how to deploy a Ruby on Rails 4.2 application on a Ubuntu 14.04 server from start to finish. I know there are lots of tutorials on the internet and even several books on deploying rails, however they are either outdated or not very clear how to do the deployment step by step. So in this series I will try to describe as clear as possible for each step.

In this first part I will describe how to install the web server and database. Here I will use Nginx, Unicorn and PostgreSQL.


# Application
To demonstrate the deployment, I will create a sample Rails 4.2 application called deploy_sample.

{% codeblock lang:bash %}
$ rails _4.2_ new deploy_sample -d postgresql
{% endcodeblock %}

And then I update the .gitignore file to exclude database.yml and secrets.yml from committing to repository.

Add the following lines to .gitignore

{% codeblock %}

/config/database.yml
/config/secrets.yml

{% endcodeblock %}

And then we copy the database.yml and secrets.yml to an example yml so let's say if your colleagues need to work on this project, they could copy the example yml back and set their configurations.

{% codeblock lang:bash %}

$ cp ./config/database.yml ./config/database.example.yml
$ cp ./config/secrets.yml ./config/secrets.example.yml

{% endcodeblock %}

Now let's init the git repository and commit our project.

{% codeblock lang:bash %}

$ git init
$ git add .
$ git commit -m 'initial commit'

{% endcodeblock %}

We will use github as our remote repository, so let's create a repository on github. And then follow its instructions, let's push the project.

{% codeblock lang:bash %}

git remote add origin https://github.com/climber2002/deploy_sample.git
git push -u origin master

{% endcodeblock %}

After that if we refresh the repository on github, we could see the content that we have committed, like the following snapshot.

{% img /images/deploy_sample_github.png %}

Now let's create some models, for simplicity I will just create a model called **Article** using scaffold.

{% codeblock lang:bash %}
bundle exec rails generate scaffold Article title content:text
{% endcodeblock %}

We can run the migrations and test the application is ok from accessing *http://localhost:3000*

{% codeblock lang:bash %}
bundle exec rake db:create
bundle exec rake db:migrate
bundle exec rails s
{% endcodeblock %}

After we can commit our changes,

{% codeblock lang:bash %}
git add .
git commit -m 'add Article scaffold'
git push origin master
{% endcodeblock %}

Now this sample application will be used for our deployment

# Vagrant
For demonstration I will use Vagrant to create a VM. But it should apply to any VPS too. Firstly you should install VirtualBox and Vagrant. The installation should be straight forward.

Then we create an empty folder in **~/Sites** and init the Vagrant.

{% codeblock lang:bash %}
mkdir ~/Sites/deploy_sample_vagrant
cd ~/Sites/deploy_sample_vagrant/
vagrant init 'ubuntu/trusty64'
{% endcodeblock %}

The vagrant init command creates a new file called Vagantfile. This file is configured to use Ubuntu 14.04 LTS server, codenamed “trusty”.

And I want to assign this VM a private IP address, so we can access it from host server later. So open the Vagrantfile and add the following line.

{% codeblock %}
config.vm.network "private_network", ip: "192.168.22.10"
{% endcodeblock %}

It assigns the IP address "192.168.22.10" to the VM. Later we can see that we can access the nginx by using this IP address.

Now in this folder we could type **vagrant up** to start the VM. If it's the first time it will take some time as it needs to download the VM image from the internet.

After the VM is up, we could type **vagrant ssh** to ssh into the VM. Notice the prompt will change. If you are not sure whether you are in host server or the VM, just check the prompt.

{% codeblock %}
vagrant@vagrant-ubuntu-trusty-64:~$
{% endcodeblock %}

# Install Basic Software
After our VM is up we could install some basic software.

So in the VM, we type following commands,

{% codeblock lang:bash %}
sudo apt-get -y update
sudo apt-get -y install curl wget unzip git ack-grep htop vim tmux software-properties-common
{% endcodeblock %}

# Nginx
Now let's install Nginx. As the default Nginx distribution may be out of date, let's add Nginx repository and install it.

{% codeblock lang:bash %}

sudo add-apt-repository ppa:nginx/stable
sudo apt-get -y update
sudo apt-get -y install nginx

{% endcodeblock %}

After installation Nginx should start automatically. We could check its status by executing **sudo service nginx status**. And we could also access [http://192.168.22.10](http://192.168.22.10) from our browser.

{% img /images/nginx.png %}

# PostgreSQL

Now let's install PostgreSQL. And again we add another repository.

{% codeblock lang:bash %}

sudo add-apt-repository ppa:pitti/postgresql
sudo apt-get -y update
sudo apt-get -y install postgresql libpq-dev

{% endcodeblock %}

The libpq-dev library is needed for building the 'pg' rubygem.

Installing PostgreSQL will create a user named 'postgres'. This user is the owner of PostgreSQL.

Let's create a password for postgres.

{% codeblock lang:bash %}
$ sudo passwd postgres
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
{% endcodeblock %}

In PostgreSQL the user is represented by a Role. Now let's create a role named *deploy_sample*.

{% codeblock lang:bash %}

$ sudo -u postgres psql

postgres=# create user deploy_sample with password 'secret';
CREATE ROLE
postgres=# create database deploy_sample_production owner deploy_sample;
CREATE DATABASE
postgres=# \q

{% endcodeblock %}

The above lines will create a user *deploy_sample* and a database *deploy_sample_production* and set deploy_sample as its owner. The last command *\q* is to exit psql.

# Monit
In next part we will use [capistrano template](https://github.com/TalkingQuickly/capistrano-3-rails-template) for our deployment. And in that template it will also deploy [Monit](http://mmonit.com/monit/). According to its official website, "Monit is a small Open Source utility for managing and monitoring Unix systems. Monit conducts automatic maintenance and repair and can execute meaningful causal actions in error situations."

So let's also install that.

{% codeblock lang:bash %}

sudo apt-get -y install monit

{% endcodeblock %}

After installation we can run **sudo service monit status** to confirm that monit service is already started.

# Nodejs
We need a Javascript runtime so we are going to install Nodejs.

{% codeblock lang:bash %}

sudo apt-get -y install nodejs

{% endcodeblock %}

# Summary
In this part we setup a VM server and installed Nginx and PostgreSQL. In next part we will see how to deploy our rails app by using capistrano.

