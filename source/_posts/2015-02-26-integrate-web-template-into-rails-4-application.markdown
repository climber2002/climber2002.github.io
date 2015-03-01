---
layout: post
title: "integrate web template into rails 4 application"
date: 2015-02-26 23:07:49 +1030
comments: true
categories: Rails bootstrap templates
---

In these days there are many free or paid web templates, especially for bootstrap templates. Recently I integrated a Bootstrap template called [Loop](https://wrapbootstrap.com/theme/loop-agency-and-personal-theme-WB053H4T4) which is bought on [WrapBootstrap](https://wrapbootstrap.com/). Here I will explain some practices and caveats about the integration.

## Location
{% img left /images/loop.png 300 300 %} 

After extract the template zip file, we can see the organization is like the  structure on the left. This template actually contains two templates, one for personal and one for Agency, and what I want is for the Agency. We can see that it puts all resource files like javascripts, CSS files and fonts file in folder *assets*. I want to keep this structure, so it would be easier to upgrade if it releases new version. The best practice for put 3rd-party template is to put it in **vendor/assets**, so I create a folder **loop** under **vendor/assets** and and copy the *assets* folder to **vendor/assets/loop**, like the following figure,

{% img left /images/loop_copied.png 300 300 %} 

Notice that I changed the *assets* folder from the original template and named it **loop**, so when we reference the files under it from browser it will create paths like */assets/loop/js/loop.js* because rails will append *assets* before the resource. This is to make the files in this template not collide with other assets.

## Create index file

If you read my blog [Using Index Files in Rails 4 Assets Pipeline](http://climber2002.github.io/blog/2014/08/17/using-index-files-in-rails-4-assets-pipeline/) before, you should know that the index files should be named **loop.css.scss** and **loop.js**, under foler **vendor/assets/loop**.

They have the following contents,

{% codeblock lang:css vendor/assets/loop/loop.css.scss %}

/*
 *= require loop/css/bootstrap
 *= require loop/css/font-awesome
 *= require loop/css/main
 */

{% endcodeblock %}


{% codeblock lang:javascript vendor/assets/loop/loop.js %}

//= require loop/js/jquery-2.1.0
//= require loop/js/bootstrap.js
//= require loop/js/plugins/scrollto/jquery.scrollTo-1.4.3.1-min
//= require loop/js/plugins/scrollto/jquery.localscroll-1.2.7-min
//= require loop/js/plugins/easing/jquery.easing.min
//= require loop/js/plugins/google-map/google-map
//= require loop/js/plugins/parallax/jquery.parallax-1.1.3
//= require loop/js/plugins/jpreloader/jpreloader.min
//= require loop/js/plugins/isotope/imagesloaded.pkgd
//= require loop/js/plugins/isotope/isotope.pkgd.min
//= require loop/js/plugins/wow/wow
//= require loop/js/plugins/flexslider/jquery.flexslider-min
//= require loop/js/plugins/magnific/jquery.magnific-popup.min
//= require loop/js/plugins/supersized/supersized.3.2.7.min
//= require loop/js/plugins/supersized/supersized.shutter.min
//= require loop/js/loop.js

{% endcodeblock %}

Basically the index files include all dependent css and javascripts, but pay atthention that their path should start with *loop* since we have put all assets in **vendor/assets/loop/loop** folder.

## Update CSS files
In the template's CSS files, there are reference to images such as background image, like following code,


{% codeblock lang:css main.css %}

body.fullscreen-image-background {
  background-image: url('../img/sliders/slider3.png?1393983498');
  background-size: cover;
  background-repeat: no-repeat;
  background-position: center center;
}

{% endcodeblock %}

Such url won't work because Rails will precompile those assets in *public/assets* folder, so we name this file to *main.css.scss*, and use **image-url** tag, like following,

{% codeblock lang:css main.css %}

body.fullscreen-image-background {
  background-image: image-url('loop/img/sliders/slider3.png');
  background-size: cover;
  background-repeat: no-repeat;
  background-position: center center;
}

{% endcodeblock %}

## Update JS files
Sometimes the JS files also reference image files, like following code,


{% codeblock lang:javascript loop.js %}

slides:   [       // Slideshow Images
              {image : 'assets/img/sliders/slider1.png', title : '<div class="hero-text"><h2 class="hero-heading">HANDCRAFTED</h2><p>Built to provide great visitor experience</p></div>', thumb : '', url : ''},
              {image : 'assets/img/sliders/slider2.png', title : '<div class="hero-text"><h2 class="hero-heading">PARALLAX</h2><p>Scrolling the page is fun with parallax background</p></div>', thumb : '', url : ''},
              {image : 'assets/img/sliders/slider3.png', title : '<div class="hero-text"><h2 class="hero-heading">BUY ONE FOR TWO</h2><p>Buy one to get both of the agency and personal theme</p></div>', thumb : '', url : ''}  
            ]

{% endcodeblock %}

For the same reason, after deploy production the JS files can't find the image files also, in such case we should depend on the asset pipeline and change the file to **loop.js.erb**, and use **image_path**.

{% codeblock lang:javascript loop.js.erb %}

slides:   [       // Slideshow Images
              {image : '<%= image_path 'loop/img/sliders/slider1.png'%>', title : '<div class="hero-text"><h2 class="hero-heading">HANDCRAFTED</h2><p>Built to provide great visitor experience</p></div>', thumb : '', url : ''},
              {image : '<%= image_path 'loop/img/sliders/slider2.png'%>', title : '<div class="hero-text"><h2 class="hero-heading">PARALLAX</h2><p>Scrolling the page is fun with parallax background</p></div>', thumb : '', url : ''},
              {image : '<%= image_path 'loop/img/sliders/slider3.png'%>', title : '<div class="hero-text"><h2 class="hero-heading">BUY ONE FOR TWO</h2><p>Buy one to get both of the agency and personal theme</p></div>', thumb : '', url : ''}  
            ]

{% endcodeblock %}

Also in our erb files, if we want to reference image files we should use **image_path**.


## Precompile configuration
By default when we run **rake assets:precompile**, Rails won't precompile the assets in vendor. So we need to add the loop template into configuration, in **application.rb**, we update like following,

{% codeblock lang:ruby application.rb %}

config.assets.precompile += %w(loop.css loop.js loop/*)

{% endcodeblock %}

So it will precompile index files and all assets in **vendor/assets/loop/loop** folder.

