---
layout: post
title: "安装rmagick出错"
date: 2012-04-06 11:46:21 +0800
comments: true
categories: ubuntu
---
报错信息

{% codeblock %}
Building native extensions.  This could take a while...
ERROR:  Error installing rmagick:
ERROR: Failed to build gem native extension.
{% endcodeblock %}


解决办法

{% codeblock lang:bash %}
$ sudo apt-get install libmagick9-dev
{% endcodeblock %}