---
layout: post
title: "Digging Rails: How Rails finds your templates Part 2"
date: 2015-02-22 21:24:30 +1030
comments: true
categories: Rails templates lookup_context resolver
---
In [last part](http://climber2002.github.io/blog/2015/02/21/how-rails-finds-your-templates-part-1/) we introduced that when Rails looks for a template, it firstly populate an options hash by calling **_normalize_render**. And then in the *render* method, it will call **render_to_body** and pass the options, like the following code,

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

The **render_to_body** will select the templated based on the values in options hash. If we check the source code of **AbstractController::Rendering#render_to_body**, it's nothing. Just as usual, it's overridden by other modules.

{% codeblock lang:ruby rails/actionpack/lib/abstract_controller/rendering.rb %}

module AbstractController

  module Rendering

    # Performs the actual template rendering.
    # :api: public
    def render_to_body(options = {})
    end

  end

end

{% endcodeblock %}

Just as last part, we could run *ApplicationController.ancestors_that_implement_instance_method* to find what classes or modules implement that method, and we will find the following,

{% codeblock lang:ruby %}

2.0.0-p353 :008 > ApplicationController.ancestors_that_implement_instance_method(:render_to_body)
 => [ActionController::Renderers, ActionController::Rendering, ActionView::Rendering, AbstractController::Rendering] 

{% endcodeblock %}

We can see three modules implement that method: *ActionController::Renderers*, *ActionController::Rendering*, and *ActionView::Rendering*. Let's look at each of them one by one.

## ActionController::Renderers#render_to_body

For *ActionController::Renderers#render_to_body* method, it registers a set of renderers, and then if the options contains the renderer key, then it will call that renderer. If no renderer is found, it just call *super*

{% codeblock lang:ruby rails/actionpack/lib/action_controller/metal/renderers.rb %}

module ActionController

  module Renderers

    def render_to_body(options)
      _render_to_body_with_renderer(options) || super
    end

    def _render_to_body_with_renderer(options)
      _renderers.each do |name|
        if options.key?(name)
          _process_options(options)
          method_name = Renderers._render_with_renderer_method_name(name)
          return send(method_name, options.delete(name), options)
        end
      end
      nil
    end

  end

end

{% endcodeblock %}

This is mainly for calling **render** and pass parameters like :json, :xml, like the following code,

{% codeblock lang:ruby %}

class ArticlesController < ApplicationController

  def index
    @articles = Articles.all
    render json: @articles
  end
end

{% endcodeblock %}

Since *:json* is a registered renderer in *ActionController::Renderers*, it will call that renderer. You can also register your own renderer by calling **ActionController::Renderers.add**.

## ActionController::Rendering#render_to_body
If in **ActionController::Renderers#render_to_body**, it doesn't find a renderer, then it will call *super*, which is **ActionController::Rendering#render_to_body**. Let's look at what this module does in the method.

{% codeblock lang:ruby rails/actionpack/lib/action_controller/metal/rendering.rb %}

module ActionController

  module Rendering

    RENDER_FORMATS_IN_PRIORITY = [:body, :text, :plain, :html]

    def render_to_body(options = {})
      super || _render_in_priorities(options) || ' '
    end

    private

    def _render_in_priorities(options)
      RENDER_FORMATS_IN_PRIORITY.each do |format|
        return options[format] if options.key?(format)
      end

      nil
    end

  end

end

{% endcodeblock %}

Notice that this method calls *super* first, it only call *_render_in_priorities* if *super* returns nothing.

In **_render_in_priorities** it searches the RENDER_FORMATS_IN_PRIORITY one by one, and return the option value if it finds the format.

In this module when it calls *super*, it is calling **ActionView::Rendering#render_to_body** method. Let's have a look.

## ActionView::Rendering#render_to_body

{% codeblock lang:ruby rails/actionview/lib/action_view/rendering.rb %}

module ActionView

  module Rendering

    def render_to_body(options = {})
      _process_options(options)
      _render_template(options)
    end

    # Returns an object that is able to render templates.
    # :api: private
    def view_renderer
      @_view_renderer ||= ActionView::Renderer.new(lookup_context)
    end

    private

      # Find and render a template based on the options given.
      # :api: private
      def _render_template(options) #:nodoc:
        variant = options[:variant]

        lookup_context.rendered_format = nil if options[:formats]
        lookup_context.variants = variant if variant

        view_renderer.render(view_context, options)
      end

  end

end

{% endcodeblock %}

It turns out that here is the the meat we are looking for. The **render_to_body** calls **_render_template**, and for the **_render_template**, it calls **view_renderer.render(view_context, options).

The *view_renderer* is an instance of **ActionView::Renderer**, and when it's initialized, it's passing a *lookup_context* object, which is an instance of **ActionView::LookupContext**. The **ActionView::LookupContext** contains all information about looking for a template based on the options. So in next part we will check this class in detail, and check how **LookupContext**, **ViewPaths**, **PathSet** work together to find the template.

