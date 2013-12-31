---
layout: default
title: Redis Activity
box: true
summary: Example system of setting up an aggregated activity stream
repo: https://github.com/SupermanScott/redis-activity-example
nav: projects
---
Redis powered activity feed with Aggregation on user\_id, activity\_type and day. Setup as 4 Celeryd tasks new\_activity, delete\_activity, follow and unfollow.

[https://github.com/SupermanScott/redis-activity-example](https://github.com/SupermanScott/redis-activity-example)

Install
-------
Download, install and run Redis
[http://redis.io/download](http://redis.io/download)

Install python requirements with Pip
{% highlight bash %}
pip install -r requirements.txt
{% endhighlight %}

Run Celeryd
{% highlight bash %}
celeryd
{% endhighlight %}

Execute some tasks

{% highlight python %}
Python 2.7.3 (default, Apr 20 2012, 22:39:59)
[GCC 4.6.3] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import time
>>> import tasks
>>> tasks.new_activity.delay(1, time.time(), 'picture')
<AsyncResult: 1cffc586-0442-4824-8c8f-7fc2c72de1ff>
>>> tasks.new_activity.delay(1, time.time(), 'like')
<AsyncResult: 20683c9f-676d-4f52-a911-b561f651bb0b>
>>> tasks.new_activity.delay(1, time.time(), 'picture')
<AsyncResult: 4e19bdc3-2c19-442a-93f9-065d22bdcdc9>
>>> tasks.follow_user.delay(1, 2)
<AsyncResult: c570c9c3-13d8-4688-ae41-0ebbdc337f45>
{% endhighlight %}
