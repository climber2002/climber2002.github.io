---
layout: post
title: "Customize Devise to support Multitenancy authentication"
date: 2015-03-29 21:13:21 +1030
comments: true
categories: rails devise subdomain authentication
---

Last year I read the book [Multitenancy with Rails](https://leanpub.com/multi-tenancy-rails) written by Ryan Bigg. In the book when he implements the authentication for the multi-tenancy, he uses [Warden](https://github.com/hassox/warden) instead of Devise, cause using Devise will need a lot of customization. And Recently I need to implement this multi-tenancy authentication for a Rails application, and I want to use Devise, so I checked the source code of Warden and Devise and found a way to customize Devise to do it.

In my application, I have two models, **Merchant** and **User**, one user can be the owner of one or more merchant, so they are a has_and_belongs_to_many relationship.

{% codeblock lang:ruby %}

class Merchant < ActiveRecord::Base

  EXCLUDED_SUBDOMAINS = %w(admin www administrator admins owner)

  validates_exclusion_of :subdomain, in: EXCLUDED_SUBDOMAINS, 
    message: "is not allowed. Please choose another subdomain"

  validates_format_of :subdomain, with: /\A[\w\-]+\Z/i, allow_blank: true,
    message: "is not allowed. Please choose another subdomain."

  validates :subdomain, presence: true, uniqueness: true

  mount_uploader :avatar, AvatarUploader

  before_validation do
    self.subdomain = subdomain.to_s.downcase
  end

  has_and_belongs_to_many :owners, class_name: 'User', join_table: 'merchants_owners'

  def owner?(user)
    return false unless user.present?
    !!owners.find_by(id: user.id)
  end

end

class User < ActiveRecord::Base

end

{% endcodeblock %}

Each merchant can access its web interface through a subdomain URL. In model **Merchant** there is a unique attribute **subdomain**, this will be used for subdomain. For example, one merchant's subdomain is *skywatch*, then the URL for this merchant is http://skywatch.xhop.pe . And for the authentication, a user can login this URL only if he is this merchant's owner. 

And in **Merchant** class we defined an instance method **owner?** of check if a user is owner of a merchant.

## Subdomain Routes Configuration
For all controllers related to a merchant, I put the controllers under a module **Merchant**. 

{% codeblock lang:ruby %}
constraints(Constraints::SubdomainRequired) do
  root 'merchant/dashboard#index', as: :merchant_root
  
  namespace :merchant do

    resources 'products'
    # and other resources
  end
end
{% endcodeblock %}

We set a constraint **Constraints::SubdomainRequired** to match merchant's routes.

{% codeblock lang:ruby lib/constraints/subdomain_required.rb %}
module Constraints
  class SubdomainRequired
    def self.matches?(request)
      request.subdomain.present? && request.subdomain != "www"
    end
  end
end
{% endcodeblock %}

So only when the request has a subdomain and the subdomain is not *www*, then it will match merchants' routes.

## Warden and Devise
Warden is a [Rack](http://rack.github.io/) middleware which provides authentication for web applications. You can register *Strategies* in Warden to define how to authenticate a user.

### Strategies

For example, the following is to add a **:password** strategy in Warden

{% codeblock lang:ruby %}
Warden::Strategies.add(:password) do

  def valid?
    params['username'] || params['password']
  end

  def authenticate!
    u = User.authenticate(params['username'], params['password'])
    u.nil? ? fail!("Could not log in") : success!(u)
  end
end
{% endcodeblock %}

When register a strategy, you should provide a name(**:password** as above), and an object or block, the object should implement two methods,

- **valid?**: It’s optional to declare a valid? method, and if you don’t declare it, the strategy will always be run. If you do declare it though, the strategy will only be tried if #valid? evaluates to true. This could be used to check if all necessary parameters are valid in the request

- **authenticate!**: This is where the work of actually authenticating the request steps in. Here’s where the logic for authenticating your requests occurs.

You have a number of request related methods available.

- request # Rack::Request object
- session # The session object for the request
- params # The parameters of the request
- env # The Rack env object

There are also a number of actions you can take in your strategy.

- halt! # halts cascading of strategies. Makes this one the last one processed
- pass # ignore this strategy. It is not required to call this and this is only sugar
- success! # Supply success! with a user object to log in a user. Causes a halt!
- fail! # Sets the strategy to fail. Causes a halt!
- redirect! # redirect to another url. You can supply it with params to be encoded and also options. Causes a halt!
- custom! # return a custom rack array to be handed back untouched. Causes a halt!
There’s a couple of misc things to do too:

headers # set headers to respond with relevant to the strategy
errors # provides access to an errors object. Here you can put in errors relating to authentication

For more information check [the Warden documents](https://github.com/hassox/warden/wiki/Strategies). 

### Scope
After configuring Strategies in Warden, you can set the strategies to scopes. Warden can use different strategies on different scopes, and the scopes are independent and not interfere each other. For example, you can configure two scopes :user and :admin, some resources are protected in :user scope and other advanced resources are protected in :admin scope. If you authenticated :user scope, it still needs authentication for :admin scope for advanced resources. For more details check [Warden Wiki](https://github.com/hassox/warden/wiki/Scopes) 

### Devise
Devise depends on Warden, and it registers a Strategy **database_authenticatable** by default, 

{% codeblock lang:ruby devise/lib/devise/strategies/database_authenticatable.rb %}

module Devise
  module Strategies
    # Default strategy for signing in a user, based on their email and password in the database.
    class DatabaseAuthenticatable < Authenticatable
      def authenticate!
        resource  = valid_password? && mapping.to.find_for_database_authentication(authentication_hash)
        encrypted = false

        if validate(resource){ encrypted = true; resource.valid_password?(password) }
          resource.after_database_authentication
          success!(resource)
        end

        mapping.to.new.password = password if !encrypted && Devise.paranoid
        fail(:not_found_in_database) unless resource
      end
    end
  end
end

Warden::Strategies.add(:database_authenticatable,
Devise::Strategies::DatabaseAuthenticatable)

{% endcodeblock %}

The **Devise::Strategies::DatabaseAuthenticatable** extends from **Devise::Strategies::Authentiatable**, which provides some methods for common authenticate behavior. If you are interested you could check Devise's source code. In above code, we can see that firstly it finds the *resource* from authentication_hash, for our case, the *resource* will be a **User** instance, and then if validate resource's password successfully, it calls *success!*, otherwise it calls *fail*. 

## Customize Devise Strategy
So for our case, we should not only check if the user's username and password are valid, but also we must check if the user is an owner of the merchant. 

Let's create class called **Devise::Strategies::DatabaseAuthenticatableForMerchantOwner**, we put this class in **lib** folder,

{% codeblock lang:ruby lib/devise/strategies/database_authenticatable_for_merchant_owner.rb %}

module Devise
  module Strategies
          
    class DatabaseAuthenticatableForMerchantOwner < Authenticatable

      # This code is mostly copied from Devise, but except the authentication of devise
      # the class includes this module should define a method 'custom_validate(resource)' to
      # provide other validates, for example if the resource is admin, or if th resource is 
      # an owner of a merchant etc..
      def authenticate!
        resource  = valid_password? && mapping.to.find_for_database_authentication(authentication_hash)
        encrypted = false

        if validate(resource){ encrypted = true; resource.valid_password?(password) } && 
          custom_validate(resource)
            resource.after_database_authentication
            success!(resource)
        else
          mapping.to.new.password = password if !encrypted && Devise.paranoid
          fail(:not_found_in_database)
          halt!
        end
      end

      def custom_validate(resource)
        merchant = get_merchant
        return false unless merchant.present?
        merchant.owner?(resource)
      end

      def get_merchant
        Merchant.find_by(subdomain: request.subdomain)
      end

    end
  end
end

{% endcodeblock %}

The **authenticate!** method is almost copied from **Devise::Strategies::DatabaseAuthenticatable**, however, after *validate(resource)* it also calls method **custom_validate**, and in **custom_validate** it will get the merchant from the request's subdomain and test if the resource(which is a user) is the merchant's owner.

PS: It's not a good practice to copy devise souce code here, I will update this blog if I find a better way.

## Register our new strategy
Now we need to register this new strategy. After we installed the Devise gem, it will create a file **config/initializers/devise.rb**, so let's update this file,

{% codeblock lang:ruby lib/devise/strategies/database_authenticatable_for_merchant_owner.rb %}
config.warden do |manager|

  require 'devise/strategies/database_authenticatable_for_merchant_owner'
  Warden::Strategies.add(:database_authenticatable_for_merchant_owner, 
    Devise::Strategies::DatabaseAuthenticatableForMerchantOwner)   

  manager.default_strategies(:scope => :user).delete :database_authenticatable
  manager.default_strategies(:scope => :user).push :database_authenticatable_for_merchant_owner 

end
{% endcodeblock %}

Here we need to require the file explicitly. And then we add the strategy :database_authenticatable_for_merchant_owner. 

And in the routes.rb, when we call 

{% codeblock lang:ruby config/routes.rb %}
Rails.application.routes.draw do
  devise_for :users
end
{% endcodeblock %}

Devise will define a scope **:user** in Warden, and it will add the :database_authenticatable in the default_strategies list. Since we want to use :database_authenticatable_for_merchant_owner strategy, we delete :database_authenticatable and push :database_authenticatable_for_merchant_owner

After this, our application will use :database_authenticatable_for_merchant_owner and support Multitenancy authentication. 