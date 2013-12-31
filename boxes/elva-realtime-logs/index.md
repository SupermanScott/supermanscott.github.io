---
layout: default
title: Elva Realtime log server
box: true
summary: Tornado based Realtime log server using python-redis-log Python module. Pushes new messages via Server-Side Events Install
nav: projects
---
Tornado based Realtime log server using python-redis-log Python module. Pushes new messages via Server-Side Events [https://github.com/SupermanScott/Elva](https://github.com/SupermanScott/Elva)

Install
--------
Download, install and run [Redis](http://redis.io/download)

Install python requirements
{% highlight bash %}
pip install -r requirements.txt
{% endhighlight %}

Run the application
{% highlight bash %}
python app.py
{% endhighlight %}

Open [localhost:8888](localhost:8888) in browser

Send some log messages
{% highlight python %}
>>> from redislog import handlers, logger
>>> l = logger.RedisLogger('my.logger')
>>> l.addHandler(handlers.RedisHandler.to("my:channel", host='localhost', port=6379, password='foobie'))
>>> l.info("I like pie")
>>> l.error("Trousers!", exc_info=True)
{% endhighlight %}
