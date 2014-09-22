---
layout: post
title: "Capybara integration tests with JQuery selectize"
date: 2014-09-22 22:30:13 +0930
comments: true
categories: capybara rails
---

Recently in one of my projects I utilized a popular JQuery plugin called [Selectize](http://brianreavis.github.io/selectize.js/). And I need to do Integration tests by using Capybara. Here is how to simulate some general events of Selectize.

Firstly I use poltergeist Javascript driver, so in my **rails_helper.rb** it has following configurations,

{% codeblock lang:ruby %}

Capybara.javascript_driver = :poltergeist

Capybara.configure do |config|
  config.match = :prefer_exact
  config.ignore_hidden_elements = false
end

{% endcodeblock %}

{% img /images/selectize.png %}

When a input box is selectized like the figure above, we can check it generates following DOM structure in our HTML,

 {% img /images/selectize_code.png %}

 The input '#properties-input' is our original input, it's set 'display: none' css property, and Selectize replaces it with another input, and the 'div.item' contains the items that we already selected, the 'div.selectize-dropdown' is shown as a dropdown when we click the input and lets us to choose another item.

 So if we needs to select an item, we could do like this,

{% codeblock lang:ruby %}

within(:xpath, "//input[@id='properties-input']/..") do 
  first('div.selectize-input').click 
  find('div.option', :text => 'Screen Size').click
end

{% endcodeblock %}


Here I use a xpath to find the parent of the input box that we selectized, and then within this parent we click the 'div.selectize-input', and the dropdown will show and we run `find('div.option', :text => 'Screen Size').click` to click the item that contains the text 'Screen Size'. 

Selectize also provides Ajax support, when you type something in the input, it could send Ajax requests to server to populate the items. So here we simulate the input,

{% codeblock lang:ruby %}

within(:xpath, "//input[@id='properties-input']/..") do 
  first('div.selectize-input input').set('u')
end

{% endcodeblock %}

This is to simulate input a 'u' character in the input, and if you configure the **load** option, it will send a Ajax request to the server to populate the items.

I've created a **SelectizeHelpers** class to simulate some common events,

{% codeblock lang:ruby %}

module SelectizeHelpers

  def selectize_click(id)
    selectize_within(id) do 
      first('div.selectize-input').click 
    end
  end

  def select_option(id, text)
    selectize_within(id) do
      first('div.selectize-input').click 
      find('div.option', :text => text).click
    end
  end

  def set_text(id, text)
    selectize_within(id) do 
      first('div.selectize-input input').set(text)
    end
  end

  def selectize_within(id)
    within(:xpath, "//input[@id='#{id}']/..") do 
      yield if block_given?
    end
  end

end

RSpec.configure do |config|
  config.include SelectizeHelpers, :type => :feature
end

{% endcodeblock %}

The `selectize_click(id)` is to click the selectize input, the `select_option` is to select an option whose text matches the **text**. And `set_text` is to set some text in the input. 

For the methods above, the **id** we should pass the id of the input that we apply the selectize.

 