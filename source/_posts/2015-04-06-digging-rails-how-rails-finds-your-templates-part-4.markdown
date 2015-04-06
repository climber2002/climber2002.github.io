---
layout: post
title: "Digging Rails: How Rails finds your templates Part 4"
date: 2015-04-06 19:08:37 +0930
comments: true
categories: Rails templates lookup_context resolver
---

In this part we will continue our exploration of how Rails finds your templates. If you need a refresh, here is [Part1](http://climber2002.github.io/blog/2015/02/21/how-rails-finds-your-templates-part-1/), [Part 2](http://climber2002.github.io/blog/2015/02/22/digging-rails-how-rails-finds-your-templates-part-2/) and [Part 3](http://climber2002.github.io/blog/2015/03/22/digging-rails-how-rails-finds-your-templates-part-3/).

In [last part](http://climber2002.github.io/blog/2015/03/22/digging-rails-how-rails-finds-your-templates-part-3/) we saw that **ActionView::LookupContext#find_template** is just a delegation to **ActionView::PathSet#find**. So let's check what **ActionView::PathSet#find** is doing.

## ActionView::PathSet

From the class name of **PathSet** we can get an idea that this class is to manage a set of pathes, where the templates are stored. Let's have a look at the definition of this class.

{% codeblock lang:ruby rails/actionview/lib/action_view/path_set.rb %}

module ActionView #:nodoc:
  # = Action View PathSet
  #
  # This class is used to store and access paths in Action View. A number of
  # operations are defined so that you can search among the paths in this
  # set and also perform operations on other +PathSet+ objects.
  #
  # A +LookupContext+ will use a +PathSet+ to store the paths in its context.
  class PathSet #:nodoc:
    include Enumerable

    attr_reader :paths

    delegate :[], :include?, :pop, :size, :each, to: :paths

    def initialize(paths = [])
      @paths = typecast paths
    end

    def find(*args)
      find_all(*args).first || raise(MissingTemplate.new(self, *args))
    end

    def find_all(path, prefixes = [], *args)
      prefixes = [prefixes] if String === prefixes
      prefixes.each do |prefix|
        paths.each do |resolver|
          templates = resolver.find_all(path, prefix, *args)
          return templates unless templates.empty?
        end
      end
      []
    end

    def exists?(path, prefixes, *args)
      find_all(path, prefixes, *args).any?
    end

    private

    def typecast(paths)
      paths.map do |path|
        case path
        when Pathname, String
          OptimizedFileSystemResolver.new path.to_s
        else
          path
        end
      end
    end
  end
end

{% endcodeblock %}

* We can see that the **initialize** method accepts a *paths* param, which is normally an array of strings. And in **initialize** method, it will call the typecast the *paths* and then store the typecasted value in *@paths* instance variable
* for each element in the *paths*, the **typecast** method will convert it to an **OptimizedFileSystemResolver** instance.
* The **find_all** method will interate each resolver instance in *paths* and return the first value which is not empty
* The **find** method just delegates to **find_all** method, if it can't find a template, it will raise *MissingTemplate* exception.

In **LookupContext** class, we can see how the **ActionView::PathSet** instance is created.

{% codeblock lang:ruby rails/actionview/lib/action_view/lookup_context.rb %}

module ActionView 
  class LookupContext

    # Helpers related to template lookup using the lookup context information.
    module ViewPaths
      attr_reader :view_paths

      # Whenever setting view paths, makes a copy so that we can manipulate them in
      # instance objects as we wish.
      def view_paths=(paths)
        @view_paths = ActionView::PathSet.new(Array(paths))
      end
    end

    include ViewPaths
  end
end

{% endcodeblock %}

The **view_paths=** method is defind in module **ActionView::LookupContext::ViewPaths** and **ActionView::LookupContext** includes this module. We can see that the when calling **view_paths=** method, an instance of **ActionView::PathSet** will be created and set to the **@view_paths** instance variable of **ActionView::LookupContext**.

So how this **view_paths=** method is called? It's in **ActionView::LookupContext#initialize** method.

{% codeblock lang:ruby rails/actionview/lib/action_view/lookup_context.rb %}

module ActionView 
  class LookupContext

    def initialize(view_paths, details = {}, prefixes = [])
      @details, @details_key = {}, nil
      @skip_default_locale = false
      @cache = true
      @prefixes = prefixes
      @rendered_format = nil

      self.view_paths = view_paths
      initialize_details(details)
    end

  end
end

{% endcodeblock %}

So the **view_paths** is passed to **ActionView::LookupContext** instances in **initialize**. And in **ActionView::ViewPaths** it initializes **ActionView::LookupContext**.


{% codeblock lang:ruby rails/actionview/lib/action_view/view_paths.rb %}

module ActionView
  module ViewPaths
    extend ActiveSupport::Concern

    included do
      class_attribute :_view_paths
      self._view_paths = ActionView::PathSet.new
      self._view_paths.freeze
    end

    # LookupContext is the object responsible to hold all information required to lookup
    # templates, i.e. view paths and details. Check ActionView::LookupContext for more
    # information.
    def lookup_context
      @_lookup_context ||=
        ActionView::LookupContext.new(self.class._view_paths, details_for_lookup, _prefixes)
    end

    def append_view_path(path)
      lookup_context.view_paths.push(*path)
    end

    def prepend_view_path(path)
      lookup_context.view_paths.unshift(*path)
    end

    module ClassMethods
      # Append a path to the list of view paths for this controller.
      #
      # ==== Parameters
      # * <tt>path</tt> - If a String is provided, it gets converted into
      #   the default view path. You may also provide a custom view path
      #   (see ActionView::PathSet for more information)
      def append_view_path(path)
        self._view_paths = view_paths + Array(path)
      end

      # Prepend a path to the list of view paths for this controller.
      #
      # ==== Parameters
      # * <tt>path</tt> - If a String is provided, it gets converted into
      #   the default view path. You may also provide a custom view path
      #   (see ActionView::PathSet for more information)
      def prepend_view_path(path)
        self._view_paths = ActionView::PathSet.new(Array(path) + view_paths)
      end

      # A list of all of the default view paths for this controller.
      def view_paths
        _view_paths
      end

  end
end

{% endcodeblock %}

This **ActionView::ViewPaths** will be included in a controller class. We can see that by default the *view_paths* is an empty array. But it provides *append_view_path* and *prepend_view_path* to add a view path at the end or the front of view paths. Notice that the **prepend_view_path** and **append_view_path** are defined both as class methods and instance methods.

So how the view path is initialized? Actually it's in **Rails::Engine** class. 

{% codeblock lang:ruby rails/railties/lib/rails/engine.rb %}

module Rails

  class Engine

    initializer :add_view_paths do
      views = paths["app/views"].existent
      unless views.empty?
        ActiveSupport.on_load(:action_controller){ prepend_view_path(views) if respond_to?(:prepend_view_path) }
        ActiveSupport.on_load(:action_mailer){ prepend_view_path(views) }
      end
    end

  end

end

{% endcodeblock %}

a initializer defines a code block which will be called during application startup. So when load :action_controller it will call **prepend_view_path** and the view_paths contains only one element, which is the **app/views** folder in your rails application.

# OptimizedFileSystemResolver

{% img left /images/resolver.png 300 500 %} 

Now understand how **ActionView::PathSet** is initialized, and we also know that in *find* method it actually delegates to the resolver. So let's see how the resolver implements the method.

The Resolver is to resolve a template when passing the details. The class relationship is as the above diagram,

We can see that by default the resolver used in **ActionView::PathSet**, which is **ActionView::OptimizedFileSystemResolver**, extends from **ActionView::FileSystemResolver**, which in terms extends from **ActionView::PathResolver**, which extends from **ActionView::Resolver**, which is the base class for all resolvers.

The base class **ActionView::Resolver** provides some common functionalities such as caching for all sub-class resolvers. However here I will focus on **ActionView::PathResolver**, which implements the logic how to find a template in a path.


{% codeblock lang:ruby rails/actionview/lib/action_view/template/resolver.rb %}

module ActionView

  class PathResolver < Resolver

    EXTENSIONS = { :locale => ".", :formats => ".", :variants => "+", :handlers => "." }
    DEFAULT_PATTERN = ":prefix/:action{.:locale,}{.:formats,}{+:variants,}{.:handlers,}"

    def initialize(pattern=nil)
      @pattern = pattern || DEFAULT_PATTERN
      super()
    end

    private

    def find_templates(name, prefix, partial, details)
      path = Path.build(name, prefix, partial)
      query(path, details, details[:formats])
    end

    def query(path, details, formats)
      query = build_query(path, details)

      template_paths = find_template_paths query

      template_paths.map { |template|
        handler, format, variant = extract_handler_and_format_and_variant(template, formats)
        contents = File.binread(template)

        Template.new(contents, File.expand_path(template), handler,
          :virtual_path => path.virtual,
          :format       => format,
          :variant      => variant,
          :updated_at   => mtime(template)
        )
      }
    end

    def find_template_paths(query)
      Dir[query].reject { |filename|
        File.directory?(filename) ||
          # deals with case-insensitive file systems.
          !File.fnmatch(query, filename, File::FNM_EXTGLOB)
      }
    end

    # Helper for building query glob string based on resolver's pattern.
    def build_query(path, details)
      query = @pattern.dup

      prefix = path.prefix.empty? ? "" : "#{escape_entry(path.prefix)}\\1"
      query.gsub!(/\:prefix(\/)?/, prefix)

      partial = escape_entry(path.partial? ? "_#{path.name}" : path.name)
      query.gsub!(/\:action/, partial)

      details.each do |ext, variants|
        query.gsub!(/\:#{ext}/, "{#{variants.compact.uniq.join(',')}}")
      end

      File.expand_path(query, @path)
    end

    def escape_entry(entry)
      entry.gsub(/[*?{}\[\]]/, '\\\\\\&')
    end
  end

end

{% endcodeblock %}

The most important methods are **build_query** and **find_template_paths**.

* The **PathResolver** will use a pattern to search templates in path, and by default the pattern is DEFAULT_PATTERN
* **build_query**: This method is to build a query from the path and details. Remember in [last part](http://climber2002.github.io/blog/2015/03/22/digging-rails-how-rails-finds-your-templates-part-3/), **ActionView::AbstractRenderer#extract_details** will extract details from options. For example, for an index action in Articles controller, the query will be 

**articles/index{.en,}{.html,}{+:variants,}{.haml,}**

For this query, in **find_template_paths** it will call **Dir[query]**, which is actually an alias to glob which is to expand the query. For example, for the above query, {p, q} matchs either p, or q. So for example for *{.en,}* it could either match .en or an empty string. So if we define a template **articles/index.html.haml**, it will match the query. For more information, consult [Ruby doc](http://ruby-doc.org/core-2.2.0/Dir.html#method-c-glob).

So for each matched template, in **query** method it will create a **Template** to wrap the template file.

## Summary
So in summary here we can see that the flow for Rails to find a template is as following,

1. Rails initializes a set of Paths for resolve templates, by default it contains only one path *app/views*
2. When an action of a controller is accessed, the render will be called either explicitly or implicitly. 
3. Rails will extract the prefixes, action and some details from the options passed to *render*,
4. Rails find the template by search in the path set using a query composed from prefixes, action and details.





