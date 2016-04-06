---
layout: post
title: Speeding up search suggestions on Google App Engine
category: Google Cloud Platform
tags: [Google App Engine, Google Compute Engine, Sphinx Search]
---
While developing [Smuge](http://www.smuge.com) on Google App Engine I faced one issue - search suggestions were too slow - delay from key pressing and displaying new results were up to few seconds(o_O). In this post I'll explain what I've made to speed them up to dozens of ms. 
[![speeding-up-search-suggestions-on-google-app-engine](http://lh3.ggpht.com/39USHo9uGCxM-ZvGji_gBvL2cwKmVI8UuPfrAavUOrSxK7mV9EPfEVgw7xnSNJW4j-lOKF2_CXkgBG77-JMflRHk=s700)](http://denisigosite.appspot.com.storage.googleapis.com/speeding-up-search-suggestions-on-google-app-engine.jpg) 
<!--more-->
# Task

When user starts typing something in the search box, a popup-window should appear and search suggestions should be displayed there. The delay between typing and displaying results should be as small as possible. Search should be performed on three indexes simultaneously - people, groups and topics.

# Sources of the delay

*   **Server-side delay.** GAE's Search API request took about 80-100 ms per index with simplest ranking algorithm. Searching on three indexes at time (largest contains about 100K records) with a bit more complex ranking took about 300-600 ms. WTF??!!
*   **Transfer delay.** Since Google App Engine servers reside in the US (at the time writing GAE in the EU avaiable only by request), it took significant time for data to travel between client and server. For example, it takes about 120 ms for request to go round up from Moscow, Russia to US (thanks to Google Fiber it is so relatively small). So, in total we have about 500-700 ms! Not a user friendly, isn't it?

So, now how I've managed them:

# Server-side delay

So I decided to move to [Sphinx Search](http://denisigosite.appspot.com/2013/12/getting-sphinx-search-to-work-with-gae/) on Google Compute Engine. Search on three indexes (and even more - read below) with way more complex ranking took just about 20-30 ms (!!!). 

The scheme is following: AJAX request from client goes to the GAE frontend, frontend itself requests Sphinx server deployed on GCE: 

[![Sphinx with GAE](http://lh4.ggpht.com/aXs1r878s1ToI2l9HU3s_68_aKEnuVuFpbjJ_0rebH_5S44-Q2qPoTrIDbISlGtZXacmo8YWCe8m0u_jK-JQ49uU=s700)](http://denisigosite.appspot.com.storage.googleapis.com/Sphinx-with-GAE.png) 

By the way, GAE > GCE > GAE overhead is pretty small - about a couple of ms. 

I have an idea to send requests directly to the GCE, but I need to set up a web server and DNS (to make requests to something like search.smuge.com rather than IP) there.

Also, I implemented server-side caching using memcache.

# Transfer delay

To reduce transfer delay I used [Google Edge Cache](http://www.denisigo.com/2013/07/how-to-reduce-gae-app-costs/). When someone requests already cached search result - he gets it in just a few dozens of milliseconds - distance to the nearest Google Datacenter is usually less than to GAE servers. But be careful with caching - setting too big caching time can lead users to get stale results. 

Again, I have an idea to somehow distribute search servers across the world and set up DNS or use another mechanism to forward user's requests to the nearest search server to reduce this delay.

# Additional optimizing

I used one trick to cut delay even more - from dozens of ms to almost nothing. I used so called pre-caching. When user searches for something, say, "la", backend gets back results not only for "la" but also for the most probable continuations - "lan", "lar", "lal" and so on. When client gets these results it caches them and when user enters next letter - with some probability client already has results and immediately displays them (also client immediately sends request for the next results): 

[![speeding-up-search-suggestions](http://lh5.ggpht.com/uvVsRwEnz98Hauaf0lXJY9OSnFQPIZ6mMBYzXQcHqLrziLpeQodS-C4V3MB5cimaC7qm9KXLsvt2eOoMonLi-Bg=s700)](http://denisigosite.appspot.com.storage.googleapis.com/speeding-up-search-suggestions.png) 

These probable continuation letters were get by scanning about of 100K people names and bio texts of Smuge database. I just went through the texts and calculated which two-letter sequences are most frequent. Here is the array I use:

``` json
tokens = {'a': [u'n', u'r', u'l', u'm', u't'],
          'c': [u'h', u'o', u'a', u'k', u'e'],
          'b': [u'e', u'r', u'a', u'o', u'i'],
          'e': [u'r', u'n', u'l', u's', u't'],
          'd': [u'a', u'e', u'o', u'i', u'r'],
          'g': [u'e', u'a', u'r', u'o', u'i'],
          'f': [u'r', u'f', u'e', u'o', u'i'],
          'i': [u'n', u'c', u'e', u'l', u's'],
          'h': [u'a', u'e', u'o', u'i', u'n'],
          'k': [u'e', u'a', u'i', u'o'],
          'j': [u'o', u'a', u'e', u'u'],
          'm': [u'a', u'i', u'e', u'o', u'c'],
          'l': [u'e', u'l', u'a', u'i', u'o'],
          'o': [u'n', u'r', u'l', u'u', u's'],
          'n': [u'e', u'a', u'n', u'd', u'i'],
          'q': [u'u'],
          'p': [u'a', u'e', u'h', u'i', u'o'],
          's': [u't', u'o', u'a', u'e', u'h'],
          'r': [u'i', u'e', u'a', u'o', u't'],
          'u': [u'r', u'l', u's', u'n', u'e'],
          't': [u'e', u'h', u'o', u't', u'i'],
          'w': [u'a', u'i', u'e', u'o'],
          'v': [u'i', u'e', u'a'],
          'y': [u'n', u'a', u'l'],
          'x': [],
          'z': [u'a']}
```

And of course the more searches you get the more statistics you get, so you can improve this probability model based on your particular data. 

This trick has some overhead for the server - say, if we have 5 probable continuations for the letter, we have to make 18 searches instead of 3 (3 indexes x (1 bare keyword search + 5 probable continuations)). But it seems like Sphinx doesn't feel the difference at all - he still gets results in 20-30 ms. Also there are some overheads in transferring size and memory consuming on client. But they are insignificant imho.
