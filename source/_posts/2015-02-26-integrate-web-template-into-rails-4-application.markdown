---
layout: post
title: "integrate web template into rails 4 application"
date: 2015-02-26 23:07:49 +1030
comments: true
categories: Rails bootstrap templates
---

In these days there are many free or paid web templates, especially for bootstrap templates. Recently I integrated a Bootstrap template called [Loop](https://wrapbootstrap.com/theme/loop-agency-and-personal-theme-WB053H4T4) which is bought on [WrapBootstrap](https://wrapbootstrap.com/). Here I will explain some practices and caveats about the integration.

## Location
{% img left /images/loop.jpg %} 

After extract the template zip file, we can see the organization is like the  structure on the left. This template actually contains two templates, one for personal and one for Agency, and what I want is for the Agency. We can see that it puts all resource files like javascripts, CSS files and fonts file in folder *assets*. I want to keep this structure, so it would be easier to upgrade if it releases new version. The best practice for put 3rd-party template is to put it in **vendor/assets**, so I create a folder **loop** under **vendor/assets** and and copy the *assets* folder to **vendor/assets/loop**, like the following figure,

{% img /images/loop_copied.jpg %} 

Notice that I keep the *assets* folder from the original template, so when we reference the files under it from browser it will create some strange path like */assets/assets/js/loop.js* because rails will append *assets* before the resource. If you want you can also rename the *assets* folder, like name it *loop*.

## Create index file

