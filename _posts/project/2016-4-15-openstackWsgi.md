---
layout: post
title: 浅析openstack中WSGI+Restful router“框架？”
description: 就算不能实习，也要继续学习
category: project
---

##引
还有2个月不到就要毕业了，公司主管希望我去实习，奈何导师项目没有结题和导师态度强硬，只好在学校努力学习公司可能会用到的知识，以防正式工作时拖团队后腿。根据初步了解，公司一部分架构为openstack+KVM+docker。虽然个人比较关注KVM和docker相关的内容，但是按照一般化的套路，还是需要了解下架构的上层原理。所以先花了几天大概看了下openstack和顺带学习了python，没想到被圏粉，话不多说，进入正题。

关于Openstack整体架构，就不在这里赘述，而其实我也只是大概了解。简单假设openstack中仅有DashBoard(Horizon)、Compute(Nova)、Network(Neutron)、Object Storage(Swift)、Image Service(Glance)、Identity(Keystone)这几个子项目组成。Openstack的一般工作方式是通过浏览器登陆DashBoard，然后通过GUI进行各种操作。在此期间DashBoard会以HTTP的形式向其它项目请求对应的服务，而其他项目之间也是通过HTTP的形式进行通讯；项目内部则通过消息总线进行通讯，openstack支持的消息总线类型中，大部分是基于AMQP的，以后会进行分析。

##WSGI(Web Server Gateway Interface) & Python Paste
在openstack中，Dashboard仅仅是运行再Web Server上的GUI，如果涉及到相关的事务处理，需要将HTTP请求转发到Application Server。而如Nova、Glance等组件则可以理解为运行在Application Server上。WSGI是Python语言中所定义的Web服务器和Web应用程序或框架之间的通用接口标准，而这里所指的Web服务器，我理解为就是Application Server。

WSGI将Web组件分为三类：
<ul>
<li>服务器(WSGI Server)，就是我理解的Application Server。负责接收HTTP请求，并封装一系列的环境变量。</li>
<li>中间件(WSGI Middleware)，负责将HTTP请求路由给不同的应用对象，并将处理结果返回给WSGI Server。从服务端看，中间件就是个WSGI应用，而从应用端看，中间件相当于服务端。</li>
<li>应用程序(WSGI Application)，是可被调用的Python对象，一般接收两个参数，environ和start_response（回调函数）。</li>
</ul>

当一个请求发送（转发）到WSGI Server时，Server会为应用端准备上下文信息和回调函数，应用端处理完后，便使用提供的回调函数返回相应的请求结果。其中中间件则作为服务端和应用端之间交互的一种包装，经过不同的中间件，便具有不同的功能，如URL分发、权限认证等，不同中间件的组合形成和WSGI的框架。虽然感觉很复杂，但Openstack中使用Paset的Deploy组件来完成WSGI服务器和应用的构建，这一切就变得简单多了。

###Paste Deploy
使用Paste Deploy的一个主要目的是用来生成WSGI Application，而通过Deploy生成应用程序则依赖于paste配置文件，形如官方给出的这种形式：

	[composite:main]
	use = egg:Paste#urlmap
	/ = home
	/blog = blog
	/wiki = wiki
	/cms = config:cms.ini

	[app:home]
	use = egg:Paste#static
	document_root = %(here)s/htdocs

	[filter-app:blog]
	use = egg:Authentication#auth
	next = blogapp
	roles = admin
	htpasswd = /home/me/users.htpasswd

	[app:blogapp]
	use = egg:BlogApp
	database = sqlite:/home/me/blog.db

	[app:wiki]
	use = call:mywiki.main:application
	database = sqlite:/home/me/wiki.db

关于具体配置文件的解读可以参考<a href="http://www.cnblogs.com/Security-Darren/p/4087587.html">详解Paste deploy</a>，在Nova组件中，配置文件位于etc/nova/api-paste.ini，有了配置文件后则进行一下调用以生成WSGI Application：
	
	##nova/wsgi.py
	465 class Loader(object):
	466     """Used to load WSGI applications from paste configurations."""
	467        
	468     def __init__(self, config_path=None):
	.......
	.......
	485        
	486     def load_app(self, name):
	487         """Return the paste URLMap wrapped WSGI application.
	488        
	489         :param name: Name of the application to load.
	490         :returns: Paste URLMap object wrapping the requested application.
	491         :raises: `nova.exception.PasteAppNotFound`
	492        
	493         """
	494         try:
	495             LOG.debug("Loading app %(name)s from %(path)s",
	496                       {'name': name, 'path': self.config_path})
	497             return deploy.loadapp("config:%s" % self.config_path, name=name)
	498         except LookupError:
	499             LOG.exception(_LE("Couldn't lookup app: %s"), name)
	500             raise exception.PasteAppNotFound(name=name, path=self.config_path)

deploy.loadapp返回值则可用作类似以下这种方式来构建WSGI服务器：

	httpd = wsgiref.simple_server('localhost', 8282, wsgi_app)  
	httpd.serve_forever()

这样就完成了应用服务器的构建了。

##RESTful API & Routes
openstack中项目都是通过RESTful API向外提供服务，什么是RESTful架构是目前最流行的一种互联网软件架构，概念请自行搜索，或参考<a href="http://www.ruanyifeng.com/blog/2011/09/restful">理解RESTful架构</a>，RESTful的核心是资源和资源上的操作，即Openstack定义了许多资源，并实现了针对资源的各种操作函数。根据上文，HTTP请求经过WSGI中间件后，可以被路由到各个“应用”，而这仅仅是将URL路由到不同的应用，而应用中必然存在许多“资源”以及“资源”对应的各种操作函数。如何将HTTP请求路由到资源对应的函数上，则不是WSGI中间件的工作。

Openstack中使用的RESTful路由模块是Routes，是对Rails on Ruby的重新实现，其使用方式可以参考<a href="http://blog.csdn.net/tantexian/article/details/37879939">wsgi-restful-routes详解</a>。

##根据Openstack进行相关分析
有了上文的基础，可以想象下Nova组件是如何响应HTTP请求的，其大致流程应该如下所述：WSGI Server接收到HTTP请求；根据Server上的Nova app生成所使用的配置文件，将HTTP请求路由到相应的“应用”上（这里指的应用就是可调用的Python对象）;根据“应用”中Routes定义的路由规则，将请求路由到相应资源的不同操作函数上。


##参考目录
[1] http://blog.csdn.net/lqk1985/article/details/2905259

[2] http://www.cnblogs.com/stli/archive/2010/11/03/1867852.html

[3] http://blog.163.com/william_djj@126/blog/static/3516650120085111193035/

[4] http://blog.csdn.net/yetyongjin/article/details/7673837
