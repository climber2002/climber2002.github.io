---
layout: post
title: "Deploy Rails Application on Ubuntu 14.04 Part 3"
date: 2015-02-16 21:43:02 +1030
comments: true
categories: postgresql deploy unicorn capistrano
---

Here is the Part 3 of Deploying Rails Application on Ubuntu 14.04, and also the final part. In this part we will explain what the Capistrano template does.

Capistrano is a remote server automation and deployment tool written in Ruby. Capistrano 3 extends Rake DSL with its own set of DSL. I recommend you read the documents on [Capistrano website](http://capistranorb.com/).

# Install Capistrano
In **Gemfile** we add the following gems,

{% codeblock lang:ruby %}
gem 'capistrano', '~> 3.1.0', group: :development

# rails specific capistrano funcitons
gem 'capistrano-rails', '~> 1.1.0', group: :development

# integrate bundler with capistrano
gem 'capistrano-bundler', group: :development

# if you are using RBENV
gem 'capistrano-rbenv', "~> 2.0", group: :development
{% endcodeblock %}

The *capistrano-rails*, *capistrano-bundler*, and *capistrano-rbenv* are capistrano plugins. Let's focus on capistrano first.

After running **bundle install**, if we run **bundle exec cap install**, it will generate following folders and files, actually when we copy the files from the capistrano template in part 2, it does the same thing.

{% codeblock %}

├──  Capfile
├── config
│   ├── deploy
│   │   ├── production.rb
│   │   └── staging.rb
│   └── deploy.rb
└── lib
    └── capistrano
            └── tasks

{% endcodeblock %}

* In Capfile, It requires all necessary files, and also it includes all customized tasks in *lib/capistrano/tasks*. 
* Normally you have a production server and a staging server. So before you deploy new features to production server, you want to deploy to staging server first and after all QA are passed. So in *config/deploy.rb*, it defines the parameters that apply to all environments, and in *config/deploy/production.rb* and *config/deploy/staging.rb* it defines the parameters that apply to the specific environment. When we run *bundle exec cap production deploy*, it will load *config/deploy.rb* first, then *config/deploy/production.rb*, and then run the **deploy** rake task.

# Understanding Capistrano

To understand Capistrano, the best way is to read its source code. Actually it's not that hard as the source code of Capistrano is pretty easy to understand.

## DSL
Capistrano defines a DSL. For those interested, can check the capistrano source code in folder *lib/capistrano/dsl*

### Set attributes
You can set some attributes by using **set**, and then later use **fetch** to get the attribute value by key. In our sample application *config/deploy.rb*, you can see it sets a lot of attributes.

The following code snippets illustrate its use.

{% codeblock lang:ruby %}

set :application, 'deploy_sample'
set :scm, :git

fetch :application # if :application key doesn't exist, it will return nil

fetch :application, 'application_name' # if the :application key doesn't exist, it will return default value 'application_name'

set_if_empty :rbenv_type, :user # set only the :rbenv_type key is not defined

{% endcodeblock %}

### Server and Roles
Think about our deployment, at least we need a database server, an application server which is unicorn, and a web server which is Nginx or Apache. Sometimes db, app and web are on the same server, sometimes they are installed on different servers. Capistrano defines these as roles. For example, in our sample application, we define one server which has three roles,

{% codeblock lang:ruby %}

server '192.168.22.10', user: 'deploy', roles: %w{web app db}, primary: true

{% endcodeblock %}

So later if we want to want to do something on the server that acts as web and app role, we could use **roles** or **release_roles** method,

For example, the following code will create directory on the server that act as web and app role,

{% codeblock lang:ruby %}

task :directories do
  on release_roles :web, :app do
    execute :mkdir, '-p', shared_path, releases_path
  end
end

{% endcodeblock %}

If the *web* and *app* role point to two different servers, each server will create the directory. If we use :all, it will get all servers.

## Rake tasks
Capistrano 3 actually is an extension of Rake tasks. So when we run **bundle exec cap production deploy**, it will run the **deploy** Rake task. 

