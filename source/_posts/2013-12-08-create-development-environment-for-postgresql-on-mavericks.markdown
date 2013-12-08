---
layout: post
title: "Create Development Environment For PostgreSQL on Mavericks"
date: 2013-12-08 22:40:04 +0800
comments: true
categories: [postgresql, rails]
---

Recently I bought a new MacBook Pro Retina with Mevericks installed. And here are the steps that I configure the development environment for PostgreSQL on this OS for my Ruby on Rails projects.

Download Postgres App
---------------------
Download the Postgres App [here](http://postgresapp.com/), which is the quickest way to install PostgreSQL on Mac OS. My version is the latest version Postgres93.

Install XCode command line tools
--------------------------------
Install XCode, and in the menu XCode -> 'Open Developer Tool' -> 'More Developer Tool', Choose the 'Command Line Tools(OS X Mavericks) for Xcode', download the dmg file and install it.

Install pg gem
--------------
Install the pg gem with the following command,
```
gem install pg -- --with-pg-config=/Applications/Postgres93.app/Contents/MacOS/bin/pg_config
```
Notice the --with-pg-config params, otherwise you may get the following errors,

```
Building native extensions.  This could take a while...
ERROR:  Error installing pg:
  ERROR: Failed to build gem native extension.

    /Users/andywang/.rvm/rubies/ruby-2.0.0-p353/bin/ruby extconf.rb
checking for pg_config... no
No pg_config... trying anyway. If building fails, please try again with
 --with-pg-config=/path/to/pg_config
checking for libpq-fe.h... no
Can't find the 'libpq-fe.h header
*** extconf.rb failed ***
Could not create Makefile due to some reason, probably lack of necessary
libraries and/or headers.  Check the mkmf.log file for more details.  You may
need configuration options.

Provided configuration options:
  --with-opt-dir
  --with-opt-include
  --without-opt-include=${opt-dir}/include
  --with-opt-lib
  --without-opt-lib=${opt-dir}/lib
  --with-make-prog
  --without-make-prog
  --srcdir=.
  --curdir
  --ruby=/Users/andywang/.rvm/rubies/ruby-2.0.0-p353/bin/ruby
  --with-pg
  --without-pg
  --with-pg-config
  --without-pg-config
  --with-pg_config
  --without-pg_config
  --with-pg-dir
  --without-pg-dir
  --with-pg-include
  --without-pg-include=${pg-dir}/include
  --with-pg-lib
  --without-pg-lib=${pg-dir}/


Gem files will remain installed in /Users/andywang/.rvm/gems/ruby-2.0.0-p353/gems/pg-0.17.0 for inspection.
Results logged to /Users/andywang/.rvm/gems/ruby-2.0.0-p353/gems/pg-0.17.0/ext/gem_make.out
```

Create role in PostgreSQL
-------------------------
Normally I want to create a specific role in the database and don't like the anonymous access. So in my database.yml, I have the following snippets,
``` yml
development:
  adapter: postgresql
  database: ebilling_development
  host: localhost
  pool: 5
  timeout: 5000
  username: ebilling
  password: ebilling
```

So here for this project I have a user ebilling whose password is also ebilling, so I created the ebilling user with the following command,
```
/Applications/Postgres93.app/Contents/MacOS/bin/createuser ebilling
```

And then open the psql console, and assign the user as superuser and password
```
andywang=# alter user ebilling superuser password 'ebilling';
```

Now after bundle install, you can create the database with the rake command,
```
rake db:create
```

