---
layout: post
title: "Digging Rails : How Rails initializes itself Part 2"
date: 2014-08-26 19:51:23 +0930
comments: true
categories: rails
---

In [last post](/blog/2014/08/24/digging-rails-how-rails-initializes-itself/) we discussed the initialization process when a Rails application starts. And we disucssed in *config/environment.rb* it calls `Rails.application.initialize!` to initialize the application. And in this blog we will have a detailed examination what has been done in this method.

Let's have a look what's in that method,

{% codeblock lang:ruby %}
# rails/railties/lib/rails/application.rb

module Rails
  class Application < Engine

    # Initialize the application passing the given group. By default, the
    # group is :default
    def initialize!(group=:default) #:nodoc:
      raise "Application has been already initialized." if @initialized
      run_initializers(group, self)
      @initialized = true
      self
    end

  end
end

{% endcodeblock %} 

we can see that it just calls the `run_initializers` method, this method is actually an instance method in a module called *Rails::Initializable*. The *Rails::Application* has following hirarchy,

{% img /images/initializable.png 200 250 %}

At the bottom is our Sample::Application class defined in config/application.rb. Its super class is Rails::Application which inherits from Rails::Engine. And Rails::Engine's parent is Rails::Railtie. The *Rails::Railtie* includes the module *Rails::Initializable*. The `run_initializers` method is defined in that module. The Railtie is to manage the initialization and configuration of each module. In Rails each module such as Active Record, Active Support has its own Railtie class which inherits from *Rails::Railtie*. After this blog you should understand the main functionality of the Railtie.

Let's have a look at the module *Rails::Initializable*. This module provides a set of methods to manage the order of initialization.

{% codeblock lang:ruby %}
module Rails
  module Initializable
    def self.included(base) #:nodoc:
      base.extend ClassMethods
    end

    def run_initializers(group=:default, *args)
      return if instance_variable_defined?(:@ran)
      initializers.tsort_each do |initializer|
        initializer.run(*args) if initializer.belongs_to?(group)
      end
      @ran = true
    end

    def initializers
      @initializers ||= self.class.initializers_for(self)
    end

    module ClassMethods
      def initializers
        @initializers ||= Collection.new
      end

      def initializers_chain
        initializers = Collection.new
        ancestors.reverse_each do |klass|
          next unless klass.respond_to?(:initializers)
          initializers = initializers + klass.initializers
        end
        initializers
      end

      def initializers_for(binding)
        Collection.new(initializers_chain.map { |i| i.bind(binding) })
      end

      def initializer(name, opts = {}, &blk)
        raise ArgumentError, "A block must be passed when defining an initializer" unless blk
        opts[:after] ||= initializers.last.name unless initializers.empty? || initializers.find { |i| i.name == opts[:before] }
        initializers << Initializer.new(name, nil, opts, &blk)
      end
    end
  end
end

{% endcodeblock %} 

If a class includes this module, it can call the class method `initializer` to register a initializer to a class instance variable `@initializers`. This method accepts three parameters: the name, an option hash which you can define the order of the initializer, and a block, this block accepts one parameter which is normally our `Rails.application` instance.

Let's check the definition of the *Initializer* class.

{% codeblock lang:ruby %}
module Rails
  module Initializable
    class Initializer
      attr_reader :name, :block

      def initialize(name, context, options, &block)
        options[:group] ||= :default
        @name, @context, @options, @block = name, context, options, block
      end

      def before
        @options[:before]
      end

      def after
        @options[:after]
      end

      def belongs_to?(group)
        @options[:group] == group || @options[:group] == :all
      end

      def run(*args)
        @context.instance_exec(*args, &block)
      end

      def bind(context)
        return self if @context
        Initializer.new(@name, context, @options, &block)
      end
    end
  end
end
{% endcodeblock %} 

In the `initialize` method, it accepts a *context* object, notice that in *Rails::Initializable*, the `initializer` class method pass context as nil. But later a initializer could call `bind` to generate a new initializer with the context bind to an object. So if a class includes Rails::Initializable, its `@initializers` class instance variable contains a list of *initializer templates*. And these initializer templates could be instantiated by bind to a context object. And then pay attention that when a class includes *Rails::Initializable* module, it also has a `initializers` instance method, this method actually gets all initializer templates from itself and its ancestors and pass itself as the context.

Let's write some classes to verify how *Rails::Initializable* works. I created a file *parent.rb* in app/models which has following contents, 

{% codeblock lang:ruby %}

# app/models/parent.rb

class Parent

  include Rails::Initializable

  initializer 'config2' do
    config2
  end

  initializer 'config1', before: 'config2' do
    config1
  end

  def config2
    puts "config2 in #{self.class}"
  end

  def config1
    puts "config1 in #{self.class}"
  end

  def inspect
    'parent'
  end
end

{% endcodeblock %} 

This class Parent is a normal class which includes *Rails::Initializable*. Two initializers are defined: *config2* and *config1*. Although we defined *config1* after *config2*, but we passed `before: 'config2'` when initialize *config1*.

Now let's open a rails console. The Parent should have a class method which returns all initializers. Let's print the names of the initializers.

{% codeblock %}
andywang@Andys-MacBook-Pro sample$ rails c
Loading development environment (Rails 4.2.0.beta1)
2.0.0-p353 :001 > Parent.initializers.map(&:name)
 => ["config2", "config1"] 

{% endcodeblock %} 

And the initializers in the class act as a template, their instance variable *@context* is nil. Let's verify that by printing the *@context* instance variable, we can see they are nil.

{% codeblock %}
2.0.0-p353 :006 >   Parent.initializers.map { |initializer| initializer.instance_variable_get('@context') }
 => [nil, nil] 
{% endcodeblock %} 

Now let's create an instance of Parent. We can see that the instance also has an instance method called *initializers*. Let's print the name,

{% codeblock %}
2.0.0-p353 :008 >   parent = Parent.new
 => parent 
2.0.0-p353 :009 > parent.initializers.map(&:name)
 => ["config2", "config1"] 
{% endcodeblock %} 

Just now we said that the initializers in the class are like a template, and for the initializers in the object instance, we can think that the initializers are *instantiated* by setting the *@context* instance variable. We can verify that by printing the instance variable of all initializers.

{% codeblock %}
2.0.0-p353 :011 >   parent.initializers.map { |initializer| initializer.instance_variable_get('@context') }
 => [parent, parent] 
{% endcodeblock %} 

We can see that the *@context* has been set to the *parent* object. 

And if we call the *run_initializers* method,

{% codeblock %}
2.0.0-p353 :013 >   parent.run_initializers
config1 in Parent
config2 in Parent
 => true 
{% endcodeblock %} 

Notice two things here, first the config1 is executed before the config2. And secondly notice why we can call the *config1* and *config2* methods in the initializer block? Because when the initializers are run, the *self* object in the block is set to the *@context* instance variable. Since our context object is *parent*, and *config1* and *config2* are instance methods of this object, of course we can call it.

Now let's create a subclass Child1 which inherits from Parent, and in this class it defines another initializer whose name is *config_in_child1*.

{% codeblock lang:ruby %}

# app/models/child1.rb

class Child1 < Parent

  initializer 'config_in_child1', after: 'config2' do
    puts "config in child1"
  end

  def inspect
    "child1"
  end
end

{% endcodeblock %} 

Let's restart our console and try to create an instance of Child1 and call *run_initializers*.

{% codeblock %}
andywang@Andys-MacBook-Pro sample$ rails c
Loading development environment (Rails 4.2.0.beta1)
2.0.0-p353 :001 > child1 = Child1.new
 => child1 
2.0.0-p353 :002 > child1.run_initializers
config1 in Child1
config2 in Child1
config in child1
 => true 
{% endcodeblock %} 

We can see that when an instance run its initializers, it will run the initializers in its ancestors first, from top to bottom. Here the two initializers *config1* and *config2* are executed first, then it's *config_in_child1* in Child1 class.

Now let's create another class Child2 also has an initializer, similar as Child1.

{% codeblock lang:ruby %}
# app/models/child2.rb

class Child2 < Parent

  initializer 'config_in_child2', after: 'config2' do
    puts "config in child2"
  end

  def inspect
    "child2"
  end
end

{% endcodeblock %} 

And then let's create one *App* class which combines one instance of Child1 and Child2.

{% codeblock lang:ruby %}
# app/models/app.rb

class App

  include Rails::Initializable

  def initialize
    @child1 = Child1.new
    @child2 = Child2.new
  end

  def initializers
    @child1.initializers + @child2.initializers
  end

end
{% endcodeblock %} 

This class has two instance variables *@child1* and *@child2*, which is an instance of Child1 and Child2 separately. This class also includes *Rails::Initializable*, but it overrides the *initializers* method to return the intializers of both *@child1* and *@child2*. Now let's open a new rails console, and create one instance of App and call `run_initializers`.

{% codeblock %}
andywang@Andys-MacBook-Pro sample$ rails c
Loading development environment (Rails 4.2.0.beta1)
2.0.0-p353 :001 > app = App.new
 => #<App:0x007fed06691540 @child1=child1, @child2=child2> 
2.0.0-p353 :002 > app.run_initializers
config1 in Child1
config1 in Child2
config2 in Child1
config2 in Child2
config in child1
config in child2
 => true 
{% endcodeblock %} 

We can see that when we run `app.run_initializers`, it runs all initializers in @child1 and @child2. And also the order is maintained. All config1 initializers are executed before config2 initializers.

The *Rails::Application* class also overrides the *initializers* method like the *App* class above. It maintains a list of Railties and run the `Rails.application.initialize!` method is called, it run all Railties' initializers.

And we could open a rails console to display all initializers and its associated binding context.

{% codeblock %}
2.0.0-p353 :003 > pp Rails.application.initializers.tsort.map {|initializer| [initializer.name, initializer.instance_variable_get("@context").class]}
[[:set_load_path, Coffee::Rails::Engine],
 [:set_load_path, Jquery::Rails::Engine],
 [:set_load_path, Turbolinks::Engine],
 [:set_load_path, WebConsole::Engine],
 [:set_load_path, Sample::Application],
 [:set_autoload_paths, Coffee::Rails::Engine],
 [:set_autoload_paths, Jquery::Rails::Engine],
 [:set_autoload_paths, Turbolinks::Engine],
 [:set_autoload_paths, WebConsole::Engine],
 [:set_autoload_paths, Sample::Application],
 [:add_routing_paths, Coffee::Rails::Engine],
 [:add_routing_paths, Jquery::Rails::Engine],
 ....
{% endcodeblock %} 


This is a very long list. Actually by default there is 108 initializers. If you are interested you can check the Rails source code to see which initializer does what. For example, the :set_load_path is to set the $LOAD_PATH global. So each engine has a chance to modify the load path. 

So after this blog you should have a general idea what Rails has done when it calls `Rails.application.initialize!`
