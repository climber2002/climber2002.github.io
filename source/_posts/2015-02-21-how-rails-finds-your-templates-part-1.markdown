---
layout: post
title: "Digging Rails: How Rails finds your templates Part 1"
date: 2015-02-21 20:35:52 +1030
comments: true
categories: Rails templates lookup_context resolver
---

Have you ever wondered for a Rails application, when you access an action in a controller, how Rails finds the template to render? For example, when the action *index* in *ArticlesController* is accessed, by default the template *app/views/articles/index.html.erb* will be selected and rendered. Recently I'm digging the source code of Rails, and I invite you to walk through some source code in ActionPack and ActionView with me. And then I will show how the Rubygem [versioncake](https://github.com/bwillis/versioncake) works by modifying the Rails configuration. 

In this first part we firstly check how the **render** works. Notice that we check Rails 4.2 source code. If you look at another version, the implementation may be slightly different.

The entry point for the render is from the **AbstractController::Rendering#render** method. The **AbstractController** is a module shared by ActionController and ActionMailer. Since these two modules share a lot of functionalities, Rails extract those same functionalities into **AbstractController**. Let's have a look at this method.

{% codeblock lang:ruby rails/actionpack/lib/abstract_controller/rendering.rb %}

module AbstractController

  module Rendering

    # Normalize arguments, options and then delegates render_to_body and
    # sticks the result in self.response_body.
    # :api: public
    def render(*args, &block)
      options = _normalize_render(*args, &block)
      self.response_body = render_to_body(options)
      _process_format(rendered_format, options) if rendered_format
      self.response_body
    end
  end

end

{% endcodeblock %}

In our controller, we could call *render* method directly. For example, we can call *render 'new'* to render the *new.html.erb* template. Or if we don't call *render* explicitly, there is a module **ActionController::ImplicitRender** which will call a default render.

{% codeblock lang:ruby rails/actionpack/lib/action_controller/metal/implicit_renderer.rb %}

module ActionController
  module ImplicitRender
    def send_action(method, *args)
      ret = super
      default_render unless performed?
      ret
    end

    def default_render(*args)
      render(*args)
    end
  end
end

{% endcodeblock %}

The send_action will be called when the action of a controller is triggered. It first calls super, then if in the action it doesn't render anything, the *performed?* will return false. So *default_render* is called. We can see that when call *default_render* it just calls *render* without any arguments.

In **AbstractController::Rendering#render** method, it firstly calls *_normalize_render* then calls *render_to_body*. The *_normalize_render* returns an options object which is a Hash. In this part we will examine the **_normalize_render** method to see how the options is generated.

Let's see how **_normalize_render** is implemented.

{% codeblock lang:ruby rails/actionpack/lib/abstract_controller/rendering.rb %}

module AbstractController

  module Rendering
    
    private

    # Normalize args and options.
    # :api: private
    def _normalize_render(*args, &block)
      options = _normalize_args(*args, &block)
      #TODO: remove defined? when we restore AP <=> AV dependency
      if defined?(request) && request && request.variant.present?
        options[:variant] = request.variant
      end
      _normalize_options(options)
      options
    end

  end

end

{% endcodeblock %}

We see that it calls *_normalize_args* and *_normalize_options* methods. The *_normalize_args* and *_normalize_options* have different purposes.

### _normalize_args
The **_normalize_args** is to convert all args into an options hash. For example when we call *render* method, we could call like this,

{% codeblock lang:ruby %}
render 'new', status: :ok
{% endcodeblock %}

Here the first argument 'new' is a String, and **_normalize_args** is responsible to put this first argument in options hash and give it an appropriate key. 

Let's see how it's implemented.

{% codeblock lang:ruby rails/actionpack/lib/abstract_controller/rendering.rb %}

module AbstractController

  module Rendering
    
    # Normalize args by converting render "foo" to render :action => "foo" and
    # render "foo/bar" to render :file => "foo/bar".
    # :api: plugin
    def _normalize_args(action=nil, options={})
      if action.is_a? Hash
        action
      else
        options
      end
    end

  end

end

{% endcodeblock %}

We see that this method by default almost does nothing, if the action is a Hash, it returns action, otherwise if action is for example, a string, it returns the second parameter which is the options hash.

Notice that ApplicationController includes some other modules which override this method, as we will see later.

### _normalize_options
The **_normalize_options** method is for the modules to include other options. For a Rails application, the ApplicationController extends from **ActionController::Base**, and **ActionController::Base** includes a lot of modules, each module could override this method and add some other options.

Let's firstly check how this method is implemented in **AbstractController::Rendering**.

{% codeblock lang:ruby rails/actionpack/lib/abstract_controller/rendering.rb %}

module AbstractController

  module Rendering

    # Normalize options.
    # :api: plugin
    def _normalize_options(options)
      options
    end
    
  end

end

{% endcodeblock %}

By default this method does nothing. But it will be overridden in other modules.

### Override _normalize_args
Rails source code is complex, one reason is because there are many modules could override other modules's methods. For example, in a ArticlesController, let's see its ancestors,

{% codeblock lang:ruby %}

$ rails c
Loading development environment (Rails 4.2.0)
2.0.0-p353 :001 > puts ArticlesController.ancestors
ArticlesController
#<Module:0x007f8f0a971800>
ApplicationController
#<Module:0x007f8f0a8f93c8>
#<Module:0x007f8f05465118>
#<Module:0x007f8f05465140>
ActionController::Base
WebConsole::ControllerHelpers
Turbolinks::XHRHeaders
Turbolinks::Cookies
Turbolinks::XDomainBlocker
Turbolinks::Redirection
ActiveRecord::Railties::ControllerRuntime
ActionDispatch::Routing::RouteSet::MountedHelpers
ActionController::ParamsWrapper
ActionController::Instrumentation
ActionController::Rescue
ActionController::HttpAuthentication::Token::ControllerMethods
ActionController::HttpAuthentication::Digest::ControllerMethods
ActionController::HttpAuthentication::Basic::ControllerMethods
ActionController::DataStreaming
ActionController::Streaming
ActionController::ForceSSL
ActionController::RequestForgeryProtection
ActionController::Flash
ActionController::Cookies
ActionController::StrongParameters
ActiveSupport::Rescuable
ActionController::ImplicitRender
ActionController::MimeResponds
ActionController::Caching
ActionController::Caching::Fragments
ActionController::Caching::ConfigMethods
AbstractController::Callbacks
ActiveSupport::Callbacks
ActionController::EtagWithTemplateDigest
ActionController::ConditionalGet
ActionController::Head
ActionController::Renderers::All
ActionController::Renderers
ActionController::Rendering
ActionView::Layouts
ActionView::Rendering
ActionController::Redirecting
ActionController::RackDelegation
ActiveSupport::Benchmarkable
AbstractController::Logger
ActionController::UrlFor
AbstractController::UrlFor
ActionDispatch::Routing::UrlFor
ActionDispatch::Routing::PolymorphicRoutes
ActionController::ModelNaming
ActionController::HideActions
ActionController::Helpers
AbstractController::Helpers
AbstractController::AssetPaths
AbstractController::Translation
AbstractController::Rendering
ActionView::ViewPaths
ActionController::Metal
AbstractController::Base
ActiveSupport::Configurable
Object
ActiveSupport::Dependencies::Loadable
PP::ObjectMixin
JSON::Ext::Generator::GeneratorMethods::Object
Kernel
BasicObject

{% endcodeblock %}

We can see that for a typical controller, it has a lot of ancestors and most of them are modules. All modules after *AbstractController::Rendering* could override its methods. I have created a gist to check which ancestors implement a method.

{% codeblock lang:ruby %}

class Module
  def ancestors_that_implement_instance_method(instance_method)
    ancestors.find_all do |ancestor|
      (ancestor.instance_methods(false) + ancestor.private_instance_methods(false)).include?(instance_method)
    end
  end
end

{% endcodeblock %}

Run the above code in a rails console, then you can call *ClassName.ancestors_that_implement_instance_method* to check what ancestors implement a method.

Let's first see what ancestors override the *_normalize_args* method,

{% codeblock lang:ruby %}

$ rails c
Loading development environment (Rails 4.2.0)
2.0.0-p353 :013 >   ArticlesController.ancestors_that_implement_instance_method(:_normalize_args)
 => [ActionController::Rendering, ActionView::Rendering, AbstractController::Rendering]

{% endcodeblock %}

Two modules override this instance method: *ActionView::Rendering* and *ActionController::Rendering*. Let's look them in order from top to down.

Let's look at *ActionView::Rendering* first,

{% codeblock lang:ruby actionview/lib/action_view/rendering.rb %}
  
module ActionView
  module Rendering
    # Normalize args by converting render "foo" to render :action => "foo" and
    # render "foo/bar" to render :template => "foo/bar".
    # :api: private
    def _normalize_args(action=nil, options={})
      options = super(action, options)
      case action
      when NilClass
      when Hash
        options = action
      when String, Symbol
        action = action.to_s
        key = action.include?(?/) ? :template : :action
        options[key] = action
      else
        options[:partial] = action
      end

      options
    end
  end
end

{% endcodeblock %}

We can see that for the first argument *action*, if it's a string and the string include '/', the key for this argument is :file, and if it doesn't include '/', the key is :action.

So if we call *render 'new'*, the options will be { action: 'new' }, if we call *render 'articles/new'*, the options will be { file: 'articles/new' }

Now let's see how *ActionController::Rendering* overrides this method

{% codeblock lang:ruby actionpack/lib/action_controller/metal/rendering.rb %}
  
module ActionController
  module Rendering
    # Normalize arguments by catching blocks and setting them on :update.
    def _normalize_args(action=nil, options={}, &blk) #:nodoc:
      options = super
      options[:update] = blk if block_given?
      options
    end
  end
end

{% endcodeblock %}

We can see that for this override, if the method is passed a block, it will set the block to options[:update]

### Override _normalize_options
Just like *_normalize_args*, let's examine what modules override *_normalize_options*. We can see following modules implement *_normalize_options*: *[ActionController::Rendering, ActionView::Layouts, ActionView::Rendering, AbstractController::Rendering]*.

Let's check *ActionView::Rendering* first,

{% codeblock lang:ruby actionpack/lib/action_controller/metal/rendering.rb %}
  
module ActionView
  module Rendering

    # Normalize options.
    # :api: private
    def _normalize_options(options)
      options = super(options)
      if options[:partial] == true
        options[:partial] = action_name
      end

      if (options.keys & [:partial, :file, :template]).empty?
        options[:prefixes] ||= _prefixes
      end

      options[:template] ||= (options[:action] || action_name).to_s
      options
    end
  end
end

{% endcodeblock %}

We can see that it addes three options by default,

* If the options[:partial] is true, then it will set options[:partial] to *action_name*, the *action_name* is just the name of the action that is triggered. For example, if the *index* action of ArticlesController is triggered, *action_name* will be *index*.
* If the options doesn't include :partial, :file or :template, it set options[:prefixes]. Let's see what's set for this :prefixes in a minute.
* It set the options[:template]. It's either passed from the arguments or just use the action_name.

So what's in options[:prefixes], let's see how *_prefixes* method is implemented.

The **AbstractController::Rendering** module includes **ActionView::ViewPaths** module. And **_prefixes** method is implemented there.

**ActionView::ViewPaths** is an important module which we will examine in more details in later parts. It's to manage the view paths for controllers. For example, by default Rails will append a view path **#{Rails.root}app/views** so the application knows to search templates in that specific view path.

For now let's just focus on **_prefixes** method.

{% codeblock lang:ruby rails/actionview/lib/action_view/view_paths.rb %}

module ActionView

  module ViewPaths
    
    # The prefixes used in render "foo" shortcuts.
    def _prefixes # :nodoc:
      self.class._prefixes
    end

  end

end

{% endcodeblock %}

The **_prefixes** just calls the class method **_prefixes**.

{% codeblock lang:ruby rails/actionview/lib/action_view/view_paths.rb %}

module ActionView

  module ViewPaths
    module ClassMethods
      def _prefixes # :nodoc:
        @_prefixes ||= begin
          return local_prefixes if superclass.abstract?

          local_prefixes + superclass._prefixes
        end
      end

      private

      # Override this method in your controller if you want to change paths prefixes for finding views.
      # Prefixes defined here will still be added to parents' <tt>._prefixes</tt>.
      def local_prefixes
        [controller_path]
      end

    end
  end

end

{% endcodeblock %}

Notice that Rails 4.2 implements that method and handles some deprecated parent prefixes. In above code I omitted that handling for clarity.

This **ActionView::ViewPaths._prefixes** calls recursively. it append local_prefixes with superclass's _prefixes. The **local_prefixes** has just one element: **controller_path**.

The **controller_path** method is very simple, it's implemented in **AbstractController::Base**. For example, for a controller *ArticlesController*, its controller_path will be *articles*, and for a controller *Articles::CommentsController*, its controller_path will be *articles/comments*

So the **_prefixes** method first gets it's parent prefixes, and then prepends the current controller_path to the front.


For example, if our application has a *ArticlesController*, in our rails controller, we can call the following code to show the *_prefixes*.

{% codeblock lang:ruby %}

$ rails c
Loading development environment (Rails 4.2.0)
2.0.0-p353 :001 > ArticlesController.new.send(:_prefixes)
 => ["articles", "application"]

{% endcodeblock %}

We see that the prefixes contains two elements: articles and application. It's because the ApplicationController extends from ActionController::Base, but ActionController::Base is abstract. 

So now we can see that for *ArticlesController#index* action, when we just call *render* without any arguments, the options will contains following elements,

* **:prefixes** : array ["articles", "application"]
* **:template** : string "index"

Now let's see next one, how **ActionView::Layouts** overrides **_normalize_options** method,

{% codeblock lang:ruby rails/actionview/lib/action_view/layouts.rb %}

module ActionView
  module Layouts

    def _normalize_options(options) # :nodoc:
      super

      if _include_layout?(options)
        layout = options.delete(:layout) { :default }
        options[:layout] = _layout_for_option(layout)
      end
    end

  end
end

{% endcodeblock %}

We can see that if the options[:layout] is not set, the default layout is :default. And then options[:layout] is set to the result returned by *_layout_for_option*.

If you are interested you could check how **_layout_for_options** is implemented. When this module searches for the layout it will search *"app/views/layouts/#{class_name.underscore}.rb"* first, if it doesn't found, then it will search super class. Since when a rails application is generated, an application.html.erb will be put in *app/views/layouts* and since by default all controllers' parent is ApplicationController, so by default this layout will be used.

Finally let's see how **ActionController::Rendering** overrides **_normalize_options**

{% codeblock lang:ruby rails/actionpack/lib/action_controller/metal/rendering.rb %}
module ActionController
  module Rendering

    # Normalize both text and status options.
    def _normalize_options(options) #:nodoc:
      _normalize_text(options)

      if options[:html]
        options[:html] = ERB::Util.html_escape(options[:html])
      end

      if options.delete(:nothing)
        options[:body] = nil
      end

      if options[:status]
        options[:status] = Rack::Utils.status_code(options[:status])
      end

      super
    end

  end
end
{% endcodeblock %}

So this method just process :html, :nothing and :status options, which is straight forward.

So finally let's see when we call *render* in *ArticlesController#index* without any arguments, the options will contains following values,

* **:prefixes** : array ["articles", "application"]
* **:template** : string "index"
* **:layout**   : A Proc when called will return "app/views/layouts/application.html.erb"

Now we know how the options is normalized. And when Rails determines which template to render, it will extract details from the options. And we will look at how Rails determines template in later parts.
