---
layout: post
title: How to reduce Google App Engine app costs
category: Google Cloud Platform
---
Here I'm collecting recipes of how to reduce costs running application on Google App Engine [POST IS BEING UPDATED from time to time].

<!--more-->

# Minimizing idle instances

There is one simple way to reduce GAE costs which is not obvious especially for newbie GAE developers. Ever mentioned that your app runs several instances even if there is almost no traffic (this way indeed is more actual for low-traffic or new apps)? 

These instances are so called "idle instances" - ones that do nothing for significant period of time but are not being shutted down to be ready to immediately serve new requests. These instances are not being shutted down since GAE's scheduler has not enough info about traffic to adjust its work. And as of result these instances might count instance hours far beyond free 28 hours quota, and this is not good for low-traffic apps. 

So, to stop such an useless usage of resources, we should tell scheduler to limit these idle instances to one instance. In such a case one idle instance will almost be running, but additional instances that would be started to handle traffic increase will be shutted down after approx. 15-20 seconds after last request served. This way we still have nice auto-scaling to handle traffic peaks but after the peak we have just only one instance and saved budget. To get this you just need to set max_idle_instances to 1 in .yaml file of your app or module:

{% highlight yaml %}
automatic_scaling:
   max_idle_instances: 1
{% endhighlight %}

# Using Google Edge Cache

All your requests to the Google services including GAE are going through special Google Frontend webservers running on the datacenter nearest to the your location. These webservers have caching capability that forms so called Google Edge Cache - a huge cache placed as near to you as possible to minimize request latency. It works just the same way other caches do (but it is Google scale and performance!) - if response to your request is already cached it is returned immediately from cache and no request is going to origin GAE server and no instance hours of your app is used. So if you have some pages that can be cached, you just need to set [standard](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html) Cache-Control header when generating response (by default no caching is allowed):

{% highlight yaml %}
Cache-Control: max-age=86400, public
{% endhighlight %}

So, we just allowed caching for 24 hours. If you're using python Webapp framework:

{% highlight yaml %}
self.response.cache_control = 'public'
self.response.cache_control.max_age = 8600
{% endhighlight %}

At the time it seems like Google doesn't charge for using this cache. But it also does not provide any cache management tools - so be careful with it - once cached, it can not be invalidated until expired.

#  Postponing queue task execution

If you use taskqueue for executing some user request-initiated job (for example, update post views counter etc), and set task to run immediately, you might find that GAE starts a new instance for handling that task  - it happens because if your latency settings are tough enough, GAE won't wait until your main request completed and starts a new instance. Since that please pay attention to one useful argument of Task constructor - "countdown". It allows you to postpone execution of your queued task to ensure your main request completed. Setting this to a couple of seconds would likely execute your task on the same instance without starting a new one. So here is the example:

{% highlight yaml %}
taskqueue.add(url='/_do_something', countdown=5)
{% endhighlight %}
