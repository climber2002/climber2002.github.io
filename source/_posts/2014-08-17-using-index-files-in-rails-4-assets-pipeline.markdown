---
layout: post
title: "Using Index Files in Rails 4 Assets Pipeline"
date: 2014-08-17 21:15:28 +0930
comments: true
categories: 
---

Recently I bought a website template from [WrapBootstrap](https://wrapbootstrap.com/theme/tshop-responsive-e-commerce-template-WB002S263) and I wanna itegrate this into a Spree website I'm working on. According to the documents of [Asset Pipeline](http://edgeguides.rubyonrails.org/asset_pipeline.html) and the [RailsCasts](http://railscasts.com/episodes/279-understanding-the-asset-pipeline). I put the stylesheets and javascripts of this template into the folder #{Rails.root}/vendor/assets/tshop. So for example, I can access a stylesheet at #{Rails.root}/vendor/assets/tshop/stylesheets/style.css from the url http://localhost:3000/assets/stylesheets/style.css

My folder structure is like following figure,

{% img /images/rails_vendor_folder.png %}

According to the [Rails documents](http://edgeguides.rubyonrails.org/asset_pipeline.html#using-index-files), I should create a file index.js and index.css under #{Rails.root}/vendor/assets/tshop, but that doesn't work for me. Actually I named the files as tshop.js and tshop.css.scss, as the figure above. So in the application.css and application.js I chould load the tshop library by calling,

{% codeblock lang:css %}
/*
 ...
 *= require tshop
 ...
 */
 {% endcodeblock %}

 and 

{% codeblock lang:javascript %}
 //= require tshop
{% endcodeblock %}

And in the tshop.css.scss, I could load the file #{Rails.root}/vendor/assets/tshop/stylesheets/style.css by using following code,

{% codeblock lang:css %}
/*
 ...
 *= require stylesheets/style
 ...
 */
 {% endcodeblock %}

 And in tshop.js, the file #{Rails.root}/vendor/assets/tshop/javascripts/script.js could be loaded by following code,

{% codeblock lang:javascript %}
 //= require javascripts/script
{% endcodeblock %}

In the templates, I need to change the background image url, for example, in style.css, 

{% codeblock lang:css %}
.parallax-image-2 {
  background: url("/assets/stylesheets/parallax/people-collage.jpg") fixed;
  background-attachment: fixed;/* IE FIX */
}
{% endcodeblock %}

This image file will be in #{Rails.root}/vendor/assets/tshop/stylesheets/parallax/people-collage.jpg.

Here are the organization I used for my Rails 4 application. The point is to use *library.css* and *library.js* to name the index files instead of index.css and index.js.



