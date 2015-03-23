---
layout: post
title: "Digging Rails: How Rails finds your templates Part 3"
date: 2015-03-22 19:37:18 +1030
comments: true
categories: Rails templates lookup_context resolver
---

Long time no see! In [last part](http://climber2002.github.io/blog/2015/02/22/digging-rails-how-rails-finds-your-templates-part-2/) we have shown that the template is rendered through **ActionView::Renderer#render** method, so in this part we will check this class in detail.

Now let's check this class,

## ActionView::Renderer

{% codeblock lang:ruby rails/actionview/lib/action_view/renderer/renderer.rb %}

module ActionView
  
  # This is the main entry point for rendering. It basically delegates
  # to other objects like TemplateRenderer and PartialRenderer which
  # actually renders the template.
  #
  # The Renderer will parse the options from the +render+ or +render_body+
  # method and render a partial or a template based on the options. The
  # +TemplateRenderer+ and +PartialRenderer+ objects are wrappers which do all
  # the setup and logic necessary to render a view and a new object is created
  # each time +render+ is called.
  class Renderer
    attr_accessor :lookup_context

    def initialize(lookup_context)
      @lookup_context = lookup_context
    end

    # Main render entry point shared by AV and AC.
    def render(context, options)
      if options.key?(:partial)
        render_partial(context, options)
      else
        render_template(context, options)
      end
    end

    # Direct accessor to template rendering.
    def render_template(context, options) #:nodoc:
      TemplateRenderer.new(@lookup_context).render(context, options)
    end
  end
end

{% endcodeblock %}

This class is initialized with a lookup_context, which is of class **ActionView::LookupContext**. And when the **render** method is called, for a normal template, the *options* has no *:partial* key, so **render_template** is called. We can see that the **render_template** method is actually creates an instance of **TemplateRenderer** and calls its **render** method.

So let's have a look at this **TemplateRenderer** class,

{% codeblock lang:ruby rails/actionview/lib/action_view/renderer/template_renderer.rb %}

module ActionView
  
  class TemplateRenderer < AbstractRenderer

    def render(context, options)
      @view    = context
      @details = extract_details(options)
      template = determine_template(options)

      prepend_formats(template.formats)

      @lookup_context.rendered_format ||= (template.formats.first || formats.first)

      render_template(template, options[:layout], options[:locals])
    end

    private

    # Determine the template to be rendered using the given options.
    def determine_template(options)
      keys = options.has_key?(:locals) ? options[:locals].keys : []

      if options.key?(:body)
        Template::Text.new(options[:body])
      elsif options.key?(:text)
        Template::Text.new(options[:text], formats.first)
      elsif options.key?(:plain)
        Template::Text.new(options[:plain])
      elsif options.key?(:html)
        Template::HTML.new(options[:html], formats.first)
      elsif options.key?(:file)
        with_fallbacks { find_template(options[:file], nil, false, keys, @details) }
      elsif options.key?(:inline)
        handler = Template.handler_for_extension(options[:type] || "erb")
        Template.new(options[:inline], "inline template", handler, :locals => keys)
      elsif options.key?(:template)
        if options[:template].respond_to?(:render)
          options[:template]
        else
          find_template(options[:template], options[:prefixes], false, keys, @details)
        end
      else
        raise ArgumentError, "You invoked render but did not give any of :partial, :template, :inline, :file, :plain, :text or :body option."
      end
    end
  end
end

{% endcodeblock %}

This class extends from **ActionView::AbstractRenderer**, and the template is determined by the **determinate_template** method, we can see that this method checks some keys in *options* hash, for example *:body*, *:text*, etc. And last one it checks the **:template** key. From [Part 1](http://climber2002.github.io/blog/2015/02/21/how-rails-finds-your-templates-part-1/) we already knew that if the **:template** key is not set explicitly, the **options[:template]** will be the action name. And **options[:prefixes]** is an array. For example, when the *index* action of *ArticlesController* is accessed, the *:template* and *:prefixes* will be,

* **:prefixes** : array [“articles”, “application”]
* **:template** : string “index”

So the template is found by calling **find_template** method, and this method is defined in super class **ActionView::AbstractRenderer**, and let's have a look at this class.

## ActionView::AbstractRenderer

{% codeblock lang:ruby rails/actionview/lib/action_view/renderer/abstract_renderer.rb %}

module ActionView
  
  class AbstractRenderer

    delegate :find_template, :template_exists?, :with_fallbacks, :with_layout_format, :formats, :to => :@lookup_context


    def initialize(lookup_context)
      @lookup_context = lookup_context
    end

    def render
      raise NotImplementedError
    end

    protected

    def extract_details(options)
      @lookup_context.registered_details.each_with_object({}) do |key, details|
        value = options[key]

        details[key] = Array(value) if value
      end
    end
  end
    
end

{% endcodeblock %}

We can see that the **find_template** method actually is delegated to **lookup_context**, we will check this method later, but remember that when **find_template** method is called, it is passed a *details* object, which is returned by the **extract_details** method.

Check the **extract_details** method above, we can see that this method returns a hash. It get the *registered_details* from the *lookup_context* and then check if the options contains the key, if yes, the result hush will be the options value for that key(tranformed to an array).

So what's in this **ActionView::LookupContext#registered_details**? Let's check that.

## ActionView::LookupContext#registered_details

{% codeblock lang:ruby rails/actionview/lib/action_view/lookup_context.rb %}

module ActionView
  
  # LookupContext is the object responsible to hold all information required to lookup
  # templates, i.e. view paths and details. The LookupContext is also responsible to
  # generate a key, given to view paths, used in the resolver cache lookup. Since
  # this key is generated just once during the request, it speeds up all cache accesses.
  class LookupContext #:nodoc:

    mattr_accessor :registered_details
    self.registered_details = []

    def self.register_detail(name, options = {}, &block)
      self.registered_details << name
      initialize = registered_details.map { |n| "@details[:#{n}] = details[:#{n}] || default_#{n}" }

      Accessors.send :define_method, :"default_#{name}", &block
      Accessors.module_eval <<-METHOD, __FILE__, __LINE__ + 1
        def #{name}
          @details.fetch(:#{name}, [])
        end

        def #{name}=(value)
          value = value.present? ? Array(value) : default_#{name}
          _set_detail(:#{name}, value) if value != @details[:#{name}]
        end

        remove_possible_method :initialize_details
        def initialize_details(details)
          #{initialize.join("\n")}
        end
      METHOD
    end

    # Holds accessors for the registered details.
    module Accessors #:nodoc:
    end

    register_detail(:locale) do
      locales = [I18n.locale]
      locales.concat(I18n.fallbacks[I18n.locale]) if I18n.respond_to? :fallbacks
      locales << I18n.default_locale
      locales.uniq!
      locales
    end

    register_detail(:formats) { ActionView::Base.default_formats || [:html, :text, :js, :css,  :xml, :json] }
    register_detail(:variants) { [] }
    register_detail(:handlers){ Template::Handlers.extensions }
  end
    
end

{% endcodeblock %}

We can see that it has a module attribute **registered_details**, which is an array, whose elements is the detail name.

For example, in a rails console we can call **ActionView::LookupContext.registered_details** to see the default detail keys.

{% img left /images/details.png %} 

We can see that by default it registered 4 details: :locale, :formats, :variants and :handlers, just as in above code, the **register_detail** is called 4 times and passed those details keys one by one. 

And also the class maintains an instance variable called **@details** which is a hash. The key of this hash is the detail key, the value normally is an array which is the detail value initialized from **initialize_details**. (We will talk about **initialize_details** later)

Each time the **register_detail** is called, it will add several instance methods to the class. For example, when passed *:locale* to the method, it will define following methods,

* **default_locale** : returns the default locale which is the result returned from the passed block.
* **locale**: which returns the locale value stored in instance variable **@details**
* **locale=**: which sets the locale value to **@details**.

And also each time the registered_detail is called, the **initialize_details** will be redefined. For example, when firstly calling **register_detail(:locale)**, the **initialize_details** will be like following,

{% codeblock lang:ruby %}

module ActionView
  
  class LookupContext #:nodoc:

    def initialize_details(details)
      @details[:locale] = details[:locale] || default_locale
    end
  end
    
end

{% endcodeblock %}

And then when **register_detail(:formats)** is called, the **initialize_details** will be redefined like following,

{% codeblock lang:ruby %}

module ActionView
  
  class LookupContext #:nodoc:

    def initialize_details(details)
      @details[:locale] = details[:locale] || default_locale
      @details[:formats] = details[:formats] || default_formats
    end
  end
    
end

{% endcodeblock %}

So the **initialize_details** is just to get the details values from *details* for each registered detail key, if the key is not found, it will set the default detail values.

## ActionView::LookupContext#find_template

Now let's check the **ActionView::LookupContext#find_template** method.

{% codeblock lang:ruby rails/actionview/lib/action_view/lookup_context.rb %}

module ActionView
  
  class LookupContext

    include ViewPaths

    # Helpers related to template lookup using the lookup context information.
    module ViewPaths
      attr_reader :view_paths, :html_fallback_for_js

      # Whenever setting view paths, makes a copy so that we can manipulate them in
      # instance objects as we wish.
      def view_paths=(paths)
        @view_paths = ActionView::PathSet.new(Array(paths))
      end

      def find(name, prefixes = [], partial = false, keys = [], options = {})
        @view_paths.find(*args_for_lookup(name, prefixes, partial, keys, options))
      end
      alias :find_template :find

      def find_all(name, prefixes = [], partial = false, keys = [], options = {})
        @view_paths.find_all(*args_for_lookup(name, prefixes, partial, keys, options))
      end

      def exists?(name, prefixes = [], partial = false, keys = [], options = {})
        @view_paths.exists?(*args_for_lookup(name, prefixes, partial, keys, options))
      end
      alias :template_exists? :exists?

      # Adds fallbacks to the view paths. Useful in cases when you are rendering
      # a :file.
      def with_fallbacks
        added_resolvers = 0
        self.class.fallbacks.each do |resolver|
          next if view_paths.include?(resolver)
          view_paths.push(resolver)
          added_resolvers += 1
        end
        yield
      ensure
        added_resolvers.times { view_paths.pop }
      end

    protected

      def args_for_lookup(name, prefixes, partial, keys, details_options) #:nodoc:
        name, prefixes = normalize_name(name, prefixes)
        details, details_key = detail_args_for(details_options)
        [name, prefixes, partial || false, details, details_key, keys]
      end

      # Compute details hash and key according to user options (e.g. passed from #render).
      def detail_args_for(options)
        return @details, details_key if options.empty? # most common path.
        user_details = @details.merge(options)

        if @cache
          details_key = DetailsKey.get(user_details)
        else
          details_key = nil
        end

        [user_details, details_key]
      end

      # Support legacy foo.erb names even though we now ignore .erb
      # as well as incorrectly putting part of the path in the template
      # name instead of the prefix.
      def normalize_name(name, prefixes) #:nodoc:
        prefixes = prefixes.presence
        parts    = name.to_s.split('/')
        parts.shift if parts.first.empty?
        name     = parts.pop

        return name, prefixes || [""] if parts.empty?

        parts    = parts.join('/')
        prefixes = prefixes ? prefixes.map { |p| "#{p}/#{parts}" } : [parts]

        return name, prefixes
      end
    end
  end
    
end

{% endcodeblock %}

The **find_template** is implemented in module **ActionView::LookupContext::ViewPaths**, and **ActionView::LookupContext** includes this module. We can see that **find_template** is just an alias of method **find**. The **find** method delegats to *@view_paths*, which is an instance of **ActionView::PathSet**. So we need to understand what **ActionView::PathSet#find** is doing.



