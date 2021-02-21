---
layout: post
title:  "Nginx + Gunicorn + Django 部署"
date:   2017-10-12 18:38:22 +0800
categories: [Server]
tags: [Nginx,Gunicorn,Django]
---

#### 前言
- 看到很多django的部署都是nginx + uwsgi 来实现，我只能说赶紧抛弃uwsgi投入gunicorn的怀抱！！！
使用uwsgi需要做复杂的配置，而gunicorn只需要很简单的配置即可启动，还能兼容eventlet, gevent, tornado, gthread, gaiohttp。

[django官网的对uwsgi介绍](https://docs.djangoproject.com/en/dev/howto/deployment/wsgi/uwsgi/) 

[django官网的对gunicorn介绍](https://docs.djangoproject.com/en/dev/howto/deployment/wsgi/gunicorn/) 

[gunicorn官网](http://docs.gunicorn.org/en/latest/install.html)

使用gunicorn启动django的命令如下：
```
gunicorn -b 127.0.0.1:8000 --worker-class=gevent yourProject.wsgi
```
--worker-class可以选择使用那个方式启动
-b 配置地址端口

#### Nginx 配置
需要写入upstream模块（在这里还能做负载均衡，但是不讲～～～）
```
upstream yourapp{
       server 127.0.0.1:8000;
}
```
修改或者添加一个location模块
```
location / {
    proxy_pass http://yourapp;
}
```
![](/img/2017-10-12-Server_Config_Nginx_Gunicorn_django_1.png)


#### Django配置
在django的项目中的setting.py的ALLOWED_HOSTS中加入yourapp，不然无法代理




