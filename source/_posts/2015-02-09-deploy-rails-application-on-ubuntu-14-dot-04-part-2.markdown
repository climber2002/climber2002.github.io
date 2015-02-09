---
layout: post
title: "Deploy Rails Application on Ubuntu 14.04 Part 2"
date: 2015-02-09 21:27:23 +1030
comments: true
categories: postgresql deploy unicorn capistrano
---

In [Part 1](http://climber2002.github.io/blog/2015/02/08/deploy-rails-application-on-ubuntu-14-dot-04/) we have installed Nginx PostgreSQL, and also Monit. In this part I will show how to deploy the application through capistrano.

In this part we will execute many commands, some will be executed in the host machine (our Laptop or iMac for example), and some will be executed in the Vagrant VM. I will show different prompt when executing. On host machine the prompt will be **host $**, and on VM the prompt will be **vagrant@vagrant-ubuntu-trusty-64:~$**, or if we su as deploy, it will be **deploy@vagrant-ubuntu-trusty-64:~$**

# Create deploy user
Just like we use an user *postgres* to manage our database, we should also use a standalone user to manage our application. We name this user *deploy*. And now let's create this user.

{% codeblock lang:bash %}
vagrant@vagrant-ubuntu-trusty-64:~$ sudo adduser --ingroup sudo deploy
Adding user `deploy' ...
Adding new user `deploy' (1002) with group `sudo' ...
Creating home directory `/home/deploy' ...
Copying files from `/etc/skel' ...
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
Changing the user information for deploy
Enter the new value, or press ENTER for the default
  Full Name []:
  Room Number []:
  Work Phone []:
  Home Phone []:
  Other []:
Is the information correct? [Y/n] Y
{% endcodeblock %}

Set a password for *deploy* user and accept all default options.

Since we already add *deploy* user to sudo group. It can run sudo commands. However the user needs to input his password. When we run the capistrano deploy command later, it will run sudo commands. So I want to update this user so he doesn't need to type password when run sodo commands. For this we need to update the **/etc/sudoers** file. It's recommended to use the application **visudo** to update it.

{% codeblock lang:bash %}

vagrant@vagrant-ubuntu-trusty-64:~$ sudo visodu

{% endcodeblock %}

Add the following line to this file, and then type *Ctrl + O* to save it as /etc/sudoers, and type *Ctrl + X* to exit.

{% codeblock lang:bash %}

deploy  ALL=(ALL:ALL) NOPASSWD:ALL

{% endcodeblock %}

Notice that since we already add *deploy* user to sudo group, this line should be added under the line **%sudo   ALL=(ALL:ALL) ALL** so it will take effect.

Our deployment organization is like following figure,

{% img /images/deploy_structure.png %}

Our host communicates with VM by SSH, and also when the VM sync code from github, it also uses SSH. So we need two public/private keypairs here.

Let's firstly generate a keypair for *deploy* user in VM. The public key for this keypair will be configured in github so the VM can download code from github.

Let's firstly su as deploy

{% codeblock lang:bash %}

vagrant@vagrant-ubuntu-trusty-64:~$ su deploy
Password:
deploy@vagrant-ubuntu-trusty-64:/home/vagrant$

{% endcodeblock %}

Now we go to the ~/.ssh folder and create a keypair.

{% codeblock lang:bash %}

deploy@vagrant-ubuntu-trusty-64:/home/vagrant$ cd ~
deploy@vagrant-ubuntu-trusty-64:~$ mkdir .ssh
deploy@vagrant-ubuntu-trusty-64:~$ cd .ssh
deploy@vagrant-ubuntu-trusty-64:~$ ssh-keygen -t rsa -b 4096 -C climber2002@gmail.com 

{% endcodeblock %}

Accept default options, and after generation two files should be created, **id_rsa** and **id_rsa.pub**.

We can run **cat id_rsa.pub** and copy its content, and then configure this key in github. 

In github, choose Settings -> SSH keys -> Add SSH key, and paste the public key, as the following snapshot.

{% img /images/github_add_ssh_key.png %}

Now we need to add the key of our host to the Vagrant VM.

{% codeblock lang:bash %}
host$ ls -l ~/.ssh
-rw-------  1 andywang  staff  1679 Jun 25  2014 id_rsa
-rw-r--r--  1 andywang  staff   403 Jun 25  2014 id_rsa.pub
{% endcodeblock %}

Normally you should already have **id_rsa** and **id_rsa.pub**. If not run the *ssh-keygen* again in host to generate the SSH key.

And then lets copy the content of id_rsa.pub and copy it to VM's **~/.ssh/authorized_keys**. If this file doesn't exist, create it.

{% codeblock lang:bash %}
deploy@vagrant-ubuntu-trusty-64:~/.ssh$ vi authorized_keys
{% endcodeblock %}

After that let's restart ssh service

{% codeblock lang:bash %}
deploy@vagrant-ubuntu-trusty-64:~/.ssh$ sudo service ssh restart
{% endcodeblock %}

Now from your host server, you should be able to ssh into the Vagrant VM.

{% codeblock lang:bash %}
host$ ssh deploy@192.168.22.10

The authenticity of host '192.168.22.10 (192.168.22.10)' can't be established.
RSA key fingerprint is b1:e4:e0:42:a1:37:0b:04:d3:88:8c:42:15:aa:00:b4.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.22.10' (RSA) to the list of known hosts.
Welcome to Ubuntu 14.04.1 LTS (GNU/Linux 3.13.0-45-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

  System information as of Mon Feb  9 12:25:47 UTC 2015

  System load:  0.0               Processes:           96
  Usage of /:   3.1% of 39.34GB   Users logged in:     2
  Memory usage: 33%               IP address for eth0: 10.0.2.15
  Swap usage:   0%                IP address for eth1: 192.168.22.10

  Graph this data and manage this system at:
    https://landscape.canonical.com/

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud


Last login: Mon Feb  9 12:25:48 2015 from 192.168.22.1
deploy@vagrant-ubuntu-trusty-64:~$ sudo ls
{% endcodeblock %}


