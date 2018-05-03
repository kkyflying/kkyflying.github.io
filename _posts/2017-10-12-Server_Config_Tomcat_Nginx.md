---
layout: post
title:  "服务器应用的配置小记录笔记"
date:   2017-10-12 11:32:05 +0800
categories: Server
tags: Tomcat,Nginx
---

#### 修改Tomcat的默认目录
在/tomcat/conf/下
vim server.xml
把原本的 appBase="webapps" ,修改成你要设置的目录
如下
![1]({{ site.url }}/img/2017-10-12-Server_Config_Tomcat_Nginx_1.png)

#### 修改nginx的默认目录
在nginx/conf/下
vim nginx.conf
要把第一个location的root都要改了，下面的location修改才会生效
![2]({{ site.url }}/img/2017-10-12-Server_Config_Tomcat_Nginx_2.png)