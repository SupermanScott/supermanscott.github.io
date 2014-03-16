---
title: Ratis - A Redis Proxy
nav: blog
layout: default
summary: A smart load balancer for Redis that properly handles transactions.
front: true
---
Just after I read this post by [Rap Genuis](http://news.rapgenius.com/James-somers-herokus-ugly-secret-annotated) I started thinking about how what it means to do smart load balancing. In most cases, the system needs to be centralized so that it knows how loaded one machine is versus all the others. The trade-off with a system like this and a trade off that the engineers at Heroku undoubtly understood very well, is this centralized load balancer becomes a single point of failure and a potential performance bottleneck. This event spur several [hammock](https://www.youtube.com/watch?v=f84n5oFoZBc) sessions for myself. I decided I wanted to try my hand at this by taking a different approach then Bamboo's routing stack or Cedar's routing stack. And I wanted to try my hand at intelligent routing for Redis.

In this post, I describe my attempt at doing intelligent routing between Redis servers. This router shares nothing between its peers but will balance its requests across multiple Redis instances sending requests to the server that has the lowest user cpu time. This attempt is not very successful, as it does add significant latency to requests to Redis servers. Using redis-benchmark going through the proxy resulted in 3x less requests per second versus running redis-benchmark against the Redis instance itself.

Solution
--------
The goal of this solution is to query each of the Redis instances and to determine how much load they are currently handling. Ratis approaches this by using a Clojure agent to query each of the servers and update the server's state. This is achieved through the following function:

{% highlight clojure linenos %}
(defn update-server-state
  "Queries the *agent* server for its current state and updates itself"
  [server]
  (when (not= 0 (:last_update server))
    (. Thread (sleep (min (+ (* (+ 1 (:failure_count server))
                                (:healthcheck_milliseconds server))
                             (rand-int 85))
                          30000))))
  (when (not (:stopping server))
    (send-off *agent* #'update-server-state))
  (calc-new-state server)
)
{% endhighlight %}

The server keeps track of its failure count to ensure that when a server is hard down it doesn't spend too much time trying to figure out if it is up again. Each server is configured to have a health check duration. This specifies how long to wait before querying the state of the server. Ratis is configured via a yml file, here is an example:

{% highlight yaml linenos %}
alpha-pool:
  port: 10000
  servers:
    - host: localhost
      port: 6379
      priority: 1
      failure_limit: 3
      healthcheck_milliseconds: 120
      connection-pool-size: 40
    - host: localhost
      port: 7777
      failure_limit: 2
      healthcheck_milliseconds: 120
      connection-pool-size: 40
beta-pool:
  port: 10001
  servers:v
    - host: localhost
      port: 6379
      priority: 1
      failure_limit: 1
      healthcheck_milliseconds: 180
      connection-pool-size: 40
{% endhighlight %}

This configuration defines two pools of servers, alpha and beta. Ratis listens on port 10000 and 10001 for each pool and routes the requests based on the cpu delta to all servers within that pool. In the alpha-pool, there are two servers running on localhost host but different ports. Ratis will automatically figure out which one is master to ensure mutations go to that server. This also means that Ratis does not care how a master is chosen, it adapts. Using Redis Sentinel or your own baked and tested logic will work just fine. Each server has a configurable 'failure_limit' which defines the number of times Ratis can fail to determine the servers state before marking the server as down. When a server is marked as down, it will no longer receive queries.

One of the main thing Ratis does is parse an incoming request to determine a few things:
1. Does this query have to go to the master ?
2. Is this query part of a transaction ?

When Ratis receives a start transaction command, which is either a WATCH or MULTI command, it opens a new connection to the current master of a given pool. This connection is then part of the ongoing tcp connection between Ratis and the request client. Each request that occurs from the client will always use the same connection. This allows for transaction semantics of Redis to work.

When the query is neither in a transaction or a master only command then it is considered slave eligible. This is defined as:

{% highlight clojure linenos %}
(defn slave-eligible?
  [client cmd]
  (and (not (redis/master-only-command? cmd))
       (not (redis/advanced-command? cmd))
       (not (:transaction-connection @client))))
{% endhighlight %}

This queries can be routed to either the slave or the master. This function does the selection of the server and proxies the response back

{% highlight clojure linenos %}
(defn respond-slave-eligible
  "Routes payload to a redis server for response"
  [cmd ch pool]
  (log/info "Routing anywhere:" cmd)
  (let [all-servers (map deref (filter active-server (:servers pool)))
        server (least-loaded all-servers)
        connection-ch (:connection-channel server)
        sender-callback (fn [connection]
                          (redis/send-to-redis-and-respond connection cmd ch
                                                           connection-ch))]
    (log/info cmd "can be sent to" (count all-servers) "for" (:name pool))
    (lamina.core/receive (:connection-channel server) sender-callback)))
{% endhighlight %}

It first gets all the servers of the pool that are active. An active server is defined as being reachable and not currently syncing with master.

{% highlight clojure linenos %}
(defn active-server
  "Returns true if the agent server is active and alive for queries"
  [server]
  (and (not (agent-error server))
       (< 0 (:last_update @server))
       (not= 0 (:loading @server))
       (or
        (= nil (:master_sync_in_progress @server))
        (= 0 (Integer/parseInt (:master_sync_in_progress @server))))
       (not (:down @server))
       true))
{% endhighlight %}

A server is picked based on the least amount of load and then a connection is pulled out of its connection pool and used to send the query to the server. The least amount of load function is define below:

{% highlight clojure linenos %}
(defn least-loaded
  "Returns the server that has the least load"
  [servers]
  (first (sort-by :cpu_delta servers)))
{% endhighlight %}

Improvements
============
So this proxy does add latency to the entire request and that is expected. It is still way to high to be really used in production. The least-loaded function should really take into account the priority assigned in the configuration file. This priority was intended to instruct a router to bias certain Redis instances over the others. This would come in handy for cross datacenter deploys. I would also really like to have other heuristics other then user cpu change. I believe it is a good proxy for how much load the server is doing. In one test with lots of keys, I ran keys * command on one server and that server received less queries then the other server within the pool. Other heuristics that might be interesting:
1. total_commands_processed
2. mem_fragmentation_ratio
3. rdb_changes_since_last_save
4. instantaneous_ops_per_sec

Summary
=======
This has been a fun on and off project of mine. It was not ever intended for production use but I am still disappointed by its latency. There are several other Redis proxies in the open that appear to add less latency this one but each of those do not appear to try and load balance 'smartly'. [Feel free to check it out here](https://github.com/SupermanScott/ratis).
