---
layout: post
title: "Calling Google Content API for shopping by using Ruby"
date: 2015-06-14 19:53:40 +0930
comments: true
categories: ruby google merchant center content API google-api-client
---

Recently I need to call Google Content API to access Google Merchant Center by using Ruby. I googled on the internet but didn't find useful links, and also the google-api-client gem doesn't contain a lot of examples. Thanks David Tzau for his [excellent blog](https://smallmeansbig.wordpress.com/2014/12/06/how-to-call-a-google-api-using-a-service-account-part-2-of-2/), however he uses Python, and in this blog I will show how to call Google Content API by using Ruby.

## Create the Client ID for a Service Account

In my application I will use Service Account, which means that the application doesn't redirect to the Google login page and asks the user to login, instead the application calls the Content API on behalf of the Service Account. This step is the same as David's blog, so I'll copy & paste here shamelessly.

So let's create a Service Account in Google Developer Console. Go to the [Google Developer Console](https://console.developers.google.com/) and navigate to the “Credentials” menu under “API’s and Auth”.  From there click on “Create new Client ID”:

{% img /images/create_service_account_1.jpg %} 

Select the “Service Account” option:

{% img /images/create_service_account_2.jpg %} 

As soon as you create the Service Account, the system will generate a Client ID, email address, and a public/private key pair.  The private key is immediately downloaded to your browser.  Save this private key safely somewhere.  There is a password that comes with this private key that you will need later.  The password is “notasecret” and is the same for every private key that is generated.

{% img /images/create_service_account_3.jpg %} 

You will notice a client ID and email address are also created for the Service Account you just created.  Make note of the email address as you will need this later:

{% img /images/create_service_account_4.jpg %} 

## Granting Permissions inside Google Merchant Center for the Client ID

This next step is very important and is not called out very clearly in Google’s documentation and is very easy to miss.   Just because you can digitally sign JWT’s and Google’s authentication server can verify your application is calling on behalf of the Service Account, it doesn’t mean your application can access just any account’s data behind the API whether that be Google Analytics, Google Merchant Center, Adwords, etc.   In this example we are going have our program call the Content API for Shopping to retrieve a product from the corresponding Google Merchant Center account.

Login to the corresponding Google Merchant Center Account and navigate to the “Users” menu under “Settings”.  Enter the Service Account email address here and grant it “Standard” access.

{% img /images/create_service_account_5.jpg %} 

## Add google-api-client Gem

To calling Google Content API, we need to add **google-api-client** Gem into our Gemfile.

{% codeblock lang:ruby %}

gem "google-api-client"

{% endcodeblock %}

And then run **bundle install**

## Get Google::APIClient

To calling Google Content API, we need an **Google::APIClient** instance. 

{% codeblock lang:ruby %}

key = Google::APIClient::PKCS12.load_key("#{Rails.root}/config/content_api.p12",
      'notasecret')
asserter = Google::APIClient::JWTAsserter.new(ENV['GOOGLE_CONTENT_API_CLIENT_ID'],
  'https://www.googleapis.com/auth/content', key)

client = Google::APIClient.new(
  :application_name => 'Your App',
  :application_version => '1.0.0')
client.authorization = asserter.authorize
content = client.discovered_api('content', 'v2')

{% endcodeblock %}

Remember the private key file we downloaded when we create the Service Account? I saved the file in #{Rails.root}/config with file name *content_api.p12* (**If your repository is an open source repository, you should not put this file in your repository as it's very sensitive**). 

On line 1, it load the key from the content_api.p12, and on line 3 it creates an asserter instance. The ENV['GOOGLE_CONTENT_API_CLIENT_ID'] is the Client ID as indicated in the following snapshot. Also you should not put this information in your repository if your repository is public. For my project I put it in a yml file and when the application loads, it will set the key/value in the yml file as environment variables.

{% img /images/client_id.png %} 

On line 6 it creates a **Google::APIClient** instance, and on line 9 it sets the authorization. 

On line 10, it calls *client.discovered_api* to get the content api, we will use this object to dictate which api we will call when we call the api, as will show in following sections.

## Calling Content API

###Request

To call Google Content API, normally we would call the **Google::APIClient#execute** method. This method accepts a Hash parameter, and the Hash normally contains the following options,

#### **:api_method**
The **:api_method** indicates which API we would like to call. Normally we set this option by calling the *content* object we got above. To know which API to call, take a look at the [Google Content API reference](https://developers.google.com/shopping-content/v2). 

For example, to call the **Accounts: insert** API, as the figure below, we should set **:api_method** to **content.accounts.insert**

{% img /images/client_api.png %} 

#### **:parameters**
The **parameters** should be a hash, it indicates the path parameters in the request URL, for example, the **Accounts: insert** API expects a *merchantId* param, so to call this API, it should set the parameters to { :merchantId => merchant_id }. Notice that the parameter key is camel case, which doesn't conform to Ruby convention.

#### **:body_object**
If the API contains a body, we should put the body in the **:body_object** option. The **:body_object** should be a Hash, what it contains depends on the specific API.

For example, to create an Account with name 'my_account', we should set the :body_object as a hash like this,

:body_object => { :name => 'my_account' }


### Response
When call the **execute** method, it returns us a response, we should call **response.data?** to check if the response contains any data. And if the API is failed, we can check response.data['error']. If the API calling is successful, we can check the documentation to see what the response data contains.For example, for Accounts Insert, if successful it will returns a [Accounts resource](https://developers.google.com/shopping-content/v2/reference/v2/accounts#resource), so we can call **response.data['id']** to get the ID of the created Account.

## Examples

Now we give some examples as following. In the examples, the client is a **Google::APIClient** instance. Normally you should wrap this *client* instance in a class, and make the methods as instance methods of the class.

### Create Merchant Center sub-Account

To create a sub-Account, and also give an email as the admin, we will call the following API,

{% codeblock lang:ruby %}

def create_merchant_center_account(parent_merchant_id, account_name, admin_email)
  client.execute(:api_method => content.accounts.insert, :parameters => { :merchantId => parent_merchant_id },
    :body_object => { :name => account_name,
      :users => [ { :emailAddress => admin_email, :admin => true }]
    })
end

{% endcodeblock %}

This method will create a new sub-Account under the parent_merchant_id, and set the admin_email as the admin of the created sub-Account.

### Delete sub-Account

To delete a sub-Account, call the method like following,

{% codeblock lang:ruby %}

def delete_merchant_center_account(parent_merchant_id, account_id)
  client.execute(:api_method => content.accounts.delete, :parameters => { :merchantId => parent_merchant_id,
    accountId: account_id })
end

{% endcodeblock %}

It will delete the sub-Account whose ID is account_id from the parent_merchant_id.

### Create a feed
To create a feed, call the API like following,

{% codeblock lang:ruby %}

def create_merchant_center_feed(merchant_id, filename, url)
  client.execute(:api_method => content.datafeeds.insert, :parameters => { :merchantId =>   merchant_id },
    :body_object => { :contentLanguage => 'en', :contentType => 'products', :fileName => filename,
      :attributeLanguage => 'en',
      :name => filename, :targetCountry => 'AU',
      :fetchSchedule => { :fetchUrl => url, :hour => 8 }
    })
end

{% endcodeblock %}

The above method will create a feed which targets Australia and the language is english. And we set the fetch schedule as 8am of the day.
