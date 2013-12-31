---
title: Elva a realtime log system
nav: blog
layout: default
summary: Expose python logging in a realtime webpage
front: true
---
I wanted to expose long running and inter-connected Python processes to a central spot without using SSH or tailing log files. Making it super easy to see what is going on under the hood.

Technologies
------------

Solution uses Redis, python-redis-log, tornado and Server-Sent Events to push the log info to Javascript. Redis was chosen because it was an existing part of my infrastructure and provided a straight forward Publish/Subscribe system. There are plenty of other PubSub systems out there and some may fit your situation better.

Python Redis Log provides the connection between Python Logging Framework and Redis and provides a rich json output.

Tornado is an asynchronous webserver that avoids the thread per request that Apache provides. This makes better use of the CPU and memory for requests that are idle awaiting new messages which is very true for this work.

This solution also uses Server-Sent Events to push the new messages to the browser. This was chosen over Websockets because Websockets are for two way communication and the problem calls for just one way (push). But the solution will work just as well with Websockets and would be less code as a Websocket handler exists in the Tornado code base where a Server-Sent event handler does not.

Problem
-------

At my current position we have a number of long running python processes that do a lot of key work behind the scenes for our website. I want to resist the urge to SSH into a machine to inspect log files and I want an easy way to determine the health of these systems quickly without having to use SSH. Providing a central spot to view all these log files aggregated together seemed like a good solution (and providing markup so that Javascript could hide certain messages as well).

Using Redis and PubSub was an easy decision. My search though for a way to connect Redis-Py and Tornado IO Loop didn't net any results. The most common solution was to spawn a thread that just listened to redis seperate from Tornado. The result of this was Ctrl-C failed to shut down the Python process. Another solution was to create a new Redis python library tied to the Tornado IO Loop. But really, seems extreme and I couldn't get those solutions to work for me. I only need the Subscribe command to be on the Tornado IO Loop not the whole thing.

Solution
--------

The solution was to create a sub class of redis.client.PubSub that created an tornado.iostream.IOStream from the socket created by Redis-Py. Then read on that stream until \r\n and process those messages. Then the server needs an Server-Sent Events RequestHandler that can push out the awaiting Javascript. This ended up being really straight forward as Server-Sent Events are really basic unlike Websockets (with all the [various](http://www.w3.org/TR/2009/WD-websockets-20090423/) [drafts](http://www.w3.org/TR/2009/WD-websockets-20091029/)...).

The PubSub handler looks like this:

{% highlight python linenos %}
class TornadoPubSub(redis.client.PubSub):
    """
    PubSub handler that uses the IOLoop from tornado to read published messages
    """
    _stream = None
    def listen(self):
        """
        Listen for messages by telling IOLoop what to call when data is there.
        """
        if not self._stream:
            socket = self.connection._sock
            self._stream = tornado.iostream.IOStream(socket)
            self._stream.read_until('\r\n', self.process_response)

    def process_response(self, data):
        """
        Called by IOLoop when data is read.
        """
        # Pretty fragile here. @TODO: figure out how to turn socket into
        # blocking until the mulk-bulk message is completely read.
        if data[0] == 'm':
            self._stream.read_bytes(2, lambda x: self._stream.read_until('}\r\n', self.read_json_message))
        else:
            self._stream.read_until('\r\n', self.process_response)

    def read_json_message(self, data):
        """
        Reads redis protocol and gets the json message
        """
        message = json.loads(data.split('\n')[-2])
        for listener in listeners:
            listener.emit(message)

        self._stream.read_until('\r\n', self.process_response)

{% endhighlight %}

The Server-Sent Event handler is also simple. It is a simple protocol that involves a Content-type header and plain text output.

{% highlight python linenos %}
class SSEHandler(tornado.web.RequestHandler):
    def initialize(self):
        self.set_header('Content-Type', 'text/event-stream')
        self.set_header('Cache-Control', 'no-cache')
    def emit(self, data, event=None):
    """
    Actually emits the data to the waiting JS
    """
    response = u''
    encoded_data = json.dumps(data)
    if event != None:
        response += u'event: ' + unicode(event).strip() + u'\n'
    response += u'data: ' + encoded_data.strip() + u'\n\n'

    self.write(response)
    self.flush()
{% endhighlight %}

Then all it takes is a simple Javascript:

{% highlight javascript linenos %}
var source = new EventSource('/events');
// Where on_message is some Javascript function to handle the message
source.addEventListener('message', on_message);
{% endhighlight %}

Then all that is missing is connection the python processes to the python redis logger. To experiment using python REPL:

{% highlight python %}
>>> from redislog import handlers, logger
>>> l = logger.RedisLogger('my.logger')
>>> l.addHandler(handlers.RedisHandler.to("my:channel", host='localhost', port=6379, password='foobie'))
>>> l.info("I like pie")
>>> l.error("Trousers!", exc_info=True)
{% endhighlight %}

Or to use logging.config.dictConfig:

{% highlight python linenos %}
import redis
import logging
import redislog
import redislog.logger
logging.setLoggerClass(redislog.logger.RedisLogger)

LOGGING = {
    'version': 1,
    'formatters': {
        'default': {
            'format': '%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        }
    },
    'handlers': {
        'redis': {
            'class': 'redislog.handlers.RedisHandler',
            'channel' : 'hermey',
            'redis_client': redis.Redis(host='localhost', port=6379),
            'level': 'DEBUG',
        },
        'some.other.log': {
            'class': 'logging.handlers.RotatingFileHandler',
            'formatter': 'default',
            'level': 'INFO',
            'filename': '/var/log/other.log',
            'backupCount': 5,
        },
    },
    'loggers': {
        'my.log': {
            'handlers': ['some.other.log', 'redis'],
            'level': 'DEBUG',
            'format': 'default',
        },
        'exceptions': {
            'handlers': ['redis'],
        },
    }
}

{% endhighlight %}

Running the Tornado Application should then create a server on localhost:8888 that prints all log statements into a table

Running the Demo
----------------

The code is pushed to Github [https://github.com/SupermanScott/Elva](https://github.com/SupermanScott/Elva). It uses all that is described in the post along with Twitter Bootstrap and mustache.js to create a decent looking interface. It buffers the messages and provides a range control to control the speed of the output. Just checkout the repo, pip install -r requirements.txt and python app.py


Issues and Improvements
-----------------------

The big issue is how fragile the PubSub client is. It only looks at the message responses from Redis. If an ERR repsonse is returned it is ignored. And it just seems really weak to errors in general. Extending this work should involved mechisims to filter the log output by error level and which logger produced the message.
