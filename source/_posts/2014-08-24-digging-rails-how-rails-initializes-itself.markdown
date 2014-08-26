---
layout: post
title: "Digging Rails - How Rails initializes itself"
date: 2014-08-24 17:35:26 +0930
comments: true
categories: Rails
---

Recently I'm reading the Rails source code and I wanna create a series of blogs to talk about it. Today I'm going to introduct the initialization process of Rails. Just notice that I use the latest Rails 4.2.0 beta1 code for analysis. If you use Rails 4.1 or earlier, the code may be different (actually it is).

bin/rails
---------

When we run *rails s* in a Rails application folder, it will call into the bin/rails script. This script has following contents,

{% codeblock lang:ruby %}
#!/usr/bin/env ruby
begin
  load File.expand_path("../spring", __FILE__)
rescue LoadError
end
APP_PATH = File.expand_path('../../config/application',  __FILE__)
require_relative '../config/boot'
require 'rails/commands'
{% endcodeblock %}

Firstly it tries to call the *spring* script which is an application preloader to speed up development by keeping our application in the background. And then it defines the APP_PATH constant, which points to the **<app_root>/config/application**, and next it requires the config/boot.rb.

config/boot.rb
--------------
The config/boot.rb has following contents,

{% codeblock lang:ruby %}
ENV['BUNDLE_GEMFILE'] ||= File.expand_path('../../Gemfile', __FILE__)

require 'bundler/setup' # Set up gems listed in the Gemfile.
{% endcodeblock %}

Since Rails uses [Bundler](http://bundler.io/) to manage the rubygems, it defines a Gemfile which declares all dependencies of the application. The `ENV['BUNDLE_GEMFILE']` is set to the path of this Gemfile, and `require 'bundler/setup'` is calling Bundler to resolve the load paths of your Gem dependencies.

rails/command.rb
----------------
The *rails* next requires 'rails/commands', and let's have a look at that file,

{% codeblock lang:ruby %}
# rails/railties/lib/rails/commands.rb

ARGV << '--help' if ARGV.empty?

aliases = {
  "g"  => "generate",
  "d"  => "destroy",
  "c"  => "console",
  "s"  => "server",
  "db" => "dbconsole",
  "r"  => "runner"
}

command = ARGV.shift
command = aliases[command] || command

require 'rails/commands/commands_tasks'

Rails::CommandsTasks.new(ARGV).run_command!(command)
{% endcodeblock %}

We can see that when we pass *s* or *server* to the *rails* script, it will set *command* to *server* and then create an instance of Rails::CommandsTasks and call its `run_command!` instance method.

rails/commands/commands_tasks.rb
--------------------------------
The rails/commands/commands_tasks defines the `Rails::CommandsTasks` class, let's check the important part of this class,

{% codeblock lang:ruby %}
# rails/railties/lib/rails/commands/commands_tasks.rb

module Rails
  class CommandsTasks
    attr_reader :argv

    COMMAND_WHITELIST = %w(plugin generate destroy console server dbconsole runner new version help)

    def initialize(argv)
      @argv = argv
    end

    def run_command!(command)
      command = parse_command(command)
      if COMMAND_WHITELIST.include?(command)
        send(command)
      else
        write_error_message(command)
      end
    end

    private

      def parse_command(command)
        case command
        when '--version', '-v'
          'version'
        when '--help', '-h'
          'help'
        else
          command
        end
      end

  end
end
{% endcodeblock %}

In the `run_command!` instance method, it checks that if the COMMAND_WHITELIST includes the command passed, then it will just invoke an instance method whose method name matches the *command* parameter. So in our case, if we passed *server* it will invoke the `server` method, which we show below,

{% codeblock lang:ruby %}
# rails/railties/lib/rails/commands/commands_tasks.rb

module Rails
  class CommandsTasks

    def server
      set_application_directory!
      require_command!("server")

      Rails::Server.new.tap do |server|
        # We need to require application after the server sets environment,
        # otherwise the --environment option given to the server won't propagate.
        require APP_PATH
        Dir.chdir(Rails.application.root)
        server.start
      end
    end

  end
end
{% endcodeblock %}

We can see that it creates a Rails::Server instance and then require our application file which is already defined in APP_PATH.

Let's first have a look at the config/application.rb

config/application.rb
---------------------
The config/application.rb has following contents,

{% codeblock lang:ruby %}
# config/application.rb
require File.expand_path('../boot', __FILE__)

require 'rails/all'

# Require the gems listed in Gemfile, including any gems
# you've limited to :test, :development, or :production.
Bundler.require(*Rails.groups)

module Sample
  class Application < Rails::Application
    # Settings in config/environments/* take precedence over those specified here.
    # Application configuration should go into files in config/initializers
    # -- all .rb files in that directory are automatically loaded.

    # Set Time.zone default to the specified zone and make Active Record auto-convert to this zone.
    # Run "rake -D time" for a list of tasks for finding time zone names. Default is UTC.
    # config.time_zone = 'Central Time (US & Canada)'

    # The default locale is :en and all translations from config/locales/*.rb,yml are auto loaded.
    # config.i18n.load_path += Dir[Rails.root.join('my', 'locales', '*.{rb,yml}').to_s]
    # config.i18n.default_locale = :de

    # For not swallow errors in after_commit/after_rollback callbacks.
    config.active_record.raise_in_transactional_callbacks = true
  end
end
{% endcodeblock %}

Firstly it requires the config/boot.rb, since above this file is already required by rails script, the require has no effect here. 

Next it requires the 'rails/all', this file has following contents, it just load the railtie of each module. Each module has its own railtie, which is to manage the initialization and configuration. We will talk about railties in another blog so I'll not explain here.

{% codeblock lang:ruby %}
# rails/railties/lib/rails/all.rb
require "rails"

%w(
  active_record
  action_controller
  action_view
  action_mailer
  active_job
  rails/test_unit
  sprockets
).each do |framework|
  begin
    require "#{framework}/railtie"
  rescue LoadError
  end
end
{% endcodeblock %}

And then next it requires the gems we defined in Gemfile by calling `Bundler.require`. And next our application class is defined. Since I named my application as *sample*, the class is in `Sample` Module.

The Sample::Application extends from Rails::Application. After this class is defined the `Rails.application` will be set to an instance of Sample::Application. Why? Let's have a look at `Rails` module.

{% codeblock lang:ruby %}
# rails/railties/lib/rails.rb

module Rails
  class << self
    @application = @app_class = nil

    attr_writer :application
    attr_accessor :app_class, :cache, :logger
    def application
      @application ||= (app_class.instance if app_class)
    end
  end
end
{% endcodeblock %}

We can see that it defines two instance variables in Rails's singleton class, *@application* and *@app_class*. And after the app_class is defined, the *Rails.application* returns the *@application* if it's already defined, if not, it will call *app_class.instance* to create an instance.

So when does the Rails.app_class is defined? Let's have a look at the Rails::Application class.

{% codeblock lang:ruby %}
# rails/railties/lib/rails/application.rb

module Rails
  class Application < Engine
    class << self
      def inherited(base)
        super
        Rails.app_class = base
      end
    end
  end
end
{% endcodeblock %}

We can see that if there is a class inherits it, it will set the Rails.app_class to the inherited class, in our case Sample::Application. So after class Sample::Application is defined, the Rails.app_class is set, and then if we call `Rails.application`, the application instance will be initialized.

Notice that the `new` method in Rails::Application is set to private, so we can only call `instance` to generate an instance. And this instance is a singleton, which is reasonable since one Rails application should have only one application instance.


Rails::Server
-------------
After the application instance is instantiated, we go back to Rails::CommandsTasks#server method, it will call `Dir.chdir(Rails.application.root)` to go to application root folder, and then call `server.start`. Let's have a look how the Rails::Server is defined.

{% codeblock lang:ruby %}
# rails/railties/lib/rails/commands/server.rb

module Rails
  class Server < ::Rack::Server

    def initialize(*)
      super
      set_environment
    end

    def start
      print_boot_information
      trap(:INT) { exit }
      create_tmp_directories
      log_to_stdout if options[:log_stdout]

      super
    ensure
      # The '-h' option calls exit before @options is set.
      # If we call 'options' with it unset, we get double help banners.
      puts 'Exiting' unless @options && options[:daemonize]
    end

    # TODO: this is no longer required but we keep it for the moment to support older config.ru files.
    def app
      @app ||= begin
        app = super
        app.respond_to?(:to_app) ? app.to_app : app
      end
    end
  end
end
{% endcodeblock %}

We can see that Rails::Server extends from ::Rack::Server. [Rack](http://rack.github.io/) is a specification which defines an interface between the web server and application. To support Rack, you need to define an *app* object, it has a `call` method, which takes the environment hash as a parameter, and returning an Array with three elements:

* The HTTP response code
* A Hash of headers
* The response body, which must respond to each 

Although this interface is very simple, you can configure middleware around the app and intercept the request and response. 

Rack is also a [rubygem](https://github.com/rack/rack). It provides a set of middleware such as logging, set content length in HTTP response, etc. And it also defines a simple DSL to configure the middleware and app, the config file has *ru* extension and Rack will load a file called *config.ru* by default.

Let's have a look at the ::Rack::Server class,

{% codeblock lang:ruby %}
module Rack
  class Server

    attr_writer :options

    def initialize(options = nil)
      @options = options
      @app = options[:app] if options && options[:app]
    end

    def app
      @app ||= options[:builder] ? build_app_from_string : build_app_and_options_from_config
    end

    def start &blk
      # ...

      server.run wrapped_app, options, &blk
    end

    def server
      @_server ||= Rack::Handler.get(options[:server]) || Rack::Handler.default(options)
    end

    private
      def build_app_and_options_from_config
        if !::File.exist? options[:config]
          abort "configuration #{options[:config]} not found"
        end

        app, options = Rack::Builder.parse_file(self.options[:config], opt_parser)
        self.options.merge! options
        app
      end

      def build_app_from_string
        Rack::Builder.new_from_string(self.options[:builder])
      end

      def wrapped_app
        @wrapped_app ||= build_app app
      end
  end
end
{% endcodeblock %}

I only leave the relevant code here and omitted some parts such as option parsing. We can see that the `start` method creates a `@_server` instance variable which is a handler represents the web server interface, it has a `run` method which accepts the *app*. The *app* is the Rack app object we mentioned above, it's an object which reponds to `call` method which accepts an environment hash and returns an array with 3 elements: the HTTP status code, response HTTP headers and the body.

The `wrapped_app` calls the `build_app` method which just wraps some default middleware around the app. And the `app` method actually sets the `@app` instance variable, which is actually read from the config.ru file (by `build_app_from_string` method).

config.ru
---------
Every rails application has a *config.ru* file at root folder. Let's have a look at that file,

{% codeblock lang:ruby %}
# This file is used by Rack-based servers to start the application.

require ::File.expand_path('../config/environment',  __FILE__)
run Rails.application
{% endcodeblock %}

This file is actually a ruby file, it first requires the *config/environment.rb* file and then calls `run Rails.application`. The `run` method is defined in Rack DSL, and it accepts the app object. So we can imagine that if we call `Rails.application.respond_to?(:call)` it should return true.

config/environment.rb
---------------------
Let's have a look what's inside config/environment.rb. 

{% codeblock lang:ruby %}
# Load the Rails application.
require File.expand_path('../application', __FILE__)

# Initialize the Rails application.
Rails.application.initialize!
{% endcodeblock %}

It just requires *config/application.rb* (which we already required) and then initialize the application by calling `Rails.application.initialize!`. After that the initialization process is finished and the `Rails.application` could be used as Rack application.

In this part we had a detailed look at the Rails initialization process and we will dig into Railties and `Rails.application.initialize!` in following series.

