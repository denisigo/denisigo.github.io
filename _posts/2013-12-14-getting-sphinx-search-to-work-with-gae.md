---
layout: post
title: Getting Sphinx Search to work with GAE
category: Google Cloud Platform
tags: [Google App Engine, Google Compute Engine, MySQL, Sphinx Search]
---
Being not satisfied with performance of GAE Search API on the web application I'm developing - some requests to just 100K documents index took up to hundreds of ms - I decided to try stand-alone search engine called Sphinx - [http://sphinxsearch.com](http://sphinxsearch.com/) ([here is the short tutorial](http://www.denisigo.com/2013/11/installing-sphinx-search-on-google-compute-engine-and-cloud-sql/)). I set up the cheapest GCE (Google Compute Engine) micro instance and installed Sphinx. Having replicated my app's Datastore data onto Cloud SQL database already set up indexing wasn't a problem (Sphinx can use SQL databases as a source for indexing). 

<!--more-->

Initial tests via SSH gave exciting results - same queries took just a few ms! It was time to connect it to my GAE app. Sphinx offers two ways of making queries - native Sphinx API and so called SphinxQL - subset of SQL based on MySQL protocol. I decided to use SphinxQL in order to use more familiar SQL syntax. 

But recommended by Google python MySQLdb client library seemed to work only with Google Cloud SQL and only via UNIX socket. Trying to connect to SphinxQL TCP port on my GCE instance via TCP socket failed with something like "cannot create socket". It seemed like MySQLdb couldn't work with custom GAE socket implementation. 

So, I tried pure python MySQL client library called PyMySQL ([https://github.com/PyMySQL/PyMySQL](https://github.com/PyMySQL/PyMySQL)). But it also not worked due to socket.recv_into method raised SocketApiNotImplementedError somewhere in pymysql's SocketIO.readinto method. So the easiest way was to apply dirty hack until GAE team implements missed method. 

Here is the part of modified pymysql/_socketio.py file:

``` python
.....
while True:
    try:
        # DSFIX
        l = len(b)
        bb = self._sock.recv(l)
        ll = len(bb)
        b[0:ll] = bb
        return ll
        # END DSFIX
        #return self._sock.recv_into(b)
    except timeout:
.....
```

So, now everything is working just perfect and I even able to connect to remote MySQL severs not only Google Cloud SQL. 

p.s. I needed things working as soon as possible so maybe I've missed something, so please correct me in comments.
