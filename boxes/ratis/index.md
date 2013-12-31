---
layout: default
title: Ratis
box: true
summary: Redis proxy to handle master slave changes
nav: projects
---
A powerful proxy for Redis that handles when a slave promotions and transactions transparently to the client. The config.yml defines the pools of servers and each server is constantly polled to assess how they are doing. A better performing server will get more of the traffic. Performance is measured from the INFO command and is based on CPU usage.

[https://github.com/SupermanScott/ratis](https://github.com/SupermanScott/ratis)

The least loaded server is chosen via:
{% highlight clojure %}
(defn least-loaded
  "Returns the server that has the least load"
  [servers]
  (first (sort-by :cpu_delta servers)))
{% endhighlight %}

Install
-------
Use [Leiningen] (http://leiningen.org/) to run the server via
{% highlight bash %}
lein run
{% endhighlight %}

Need to configure the pools by adding the servers to the config.yml. Example below:
{% highlight yaml %}
alpha-pool:
  port: 10000
  servers:
    - host: "localhost"
      port: 6379
      priority: 1
      failure_limit: 3
      healthcheck_milliseconds: 120
    - host: "localhost"
      port: 7777
      failure_limit: 2
      healthcheck_milliseconds: 120
beta-pool:
  port: 10001
  servers:
    - host: "localhost"
      port: 6379
      priority: 1
      failure_limit: 1
      healthcheck_milliseconds: 180
{% endhighlight %}
