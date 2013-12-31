---
layout: default
title: Arya Search Engine
box: true
summary: Mongodb based search engine
nav: projects
---
This project was a way to explore Mongodb a bit and its Map Reduce system. Seemed like a fun project and it is far from complete or really that interesting. It has a basic Indexer that, given a url creates a document in Mongo and then indexes the content of the page and makes it searchable. The search provides a simple interface to run the query.

[https://github.com/SupermanScott/Arya](https://github.com/SupermanScott/Arya)

Install
-------
Download, install and run Mongodb
[http://www.mongodb.org/downloads](http://www.mongodb.org/downloads)

{% highlight bash %}
pip install -r requirements.txt
{% endhighlight %}


With that installed you are now ready to interact with it via the python console

{% highlight python %}
>>> import indexing.indexer as indexer
>>> import searchers.searcher as s
>>> idx = indexer.Indexer()
>>> idx.index_url('http://www.mongodb.org/display/DOCS/Philosophy')
>>> idx.index_url('http://www.mongodb.org/display/DOCS/Use+Cases')
>>> [(x['document']['title'], x['score']) for x in s.Searcher().search('cloud')]
>>> [(x['document']['title'], x['score']) for x in s.Searcher().search('database power')]
{% endhighlight %}

That is it! Just a small fun project.
