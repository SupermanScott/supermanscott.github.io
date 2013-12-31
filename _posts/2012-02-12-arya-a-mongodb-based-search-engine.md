---
title: Arya a Mongodb based search engine
nav: blog
layout: default
summary: Search engine powered by Mongodb
---
I wanted to explore MongoDB Map Reduce framework and to build something non-trivial So this is a system that provides an Indexer and a Searcher to do the tasks and store the data in MongoDB. It is realtime search as the index is just a collection in MongoDB.

Solution
--------

The solution uses Python and pymongo to store the data. It uses the port of Readability's javascript bookmarklet to extract the content of a given page. A tokenizer that splits on whitespace and one analyzer that runs the Porter Stem algorithm on the text. This choice means it really only works on English texts. The system does allow for multiple analyzers, the obvious case would be to have a Stop word analyzer to remove the obvious terms like 'the'. The order of the analyzers do matter. The indexing phase goes like this snippet

{% highlight python linenos %}
for token in self.tokenizer(doc.get(field_name, '')):
    original_word = token
    for analyzer in self.analyzers:
        token = analyzer(token)
    processed_tokens[token] = processed_tokens.get(token, 0) + 1
{% endhighlight %}

This snippet calculates the number of times that term shows up in that field of the document. This is then added to the reverse index for that term

{% highlight python linenos %}
term_document = dict(
    term=token,
    matches=[dict(
        doc_id=document_id,
        field_name=field_name,
        original_word=original_word,
        term_fq=frequency)],
    document_fq=1)
{% endhighlight %}

Structuring the document like this allows for a number of different things to occur. First it is important to add an index on the 'term' field in the proper collection. This will make querying by that term fast. The matches array of embedded documents provides information about the match which includes, the field that the match was found in, the original word which is different from the term, and the number of times that term was seen in that field. Combining the last property with a global count of the number of documents that contain this term gives allows for the tf\*idf score to be calculated.


Searching
---------

Searching must be accomplished by applying the same tokenizer and analyzer pipeline on the query string. Doing this ensures that when I search for "Redis" the mongo query will use the term "redi". So the query is processed by the same tokenizer and analyzer These tokens derived from the query are then used to provide a 'query' to Map Reduce operation. Here is the map function

{% highlight python linenos %}
map_function = bson.Code(
   "function () {"
   " for (var i=0; i < this.matches.length; i++) {"
   "   var match = this.matches[i];"
   "   var data = {tf: match.term_fq, df: this.document_fq};"
   "   emit(this.matches[i].doc_id, data);"
" }}")

{% endhighlight %}

The map function's job is to emit all of the documents in the term matches along with the match's term frequency and the term's document frequency. These numbers will be used to calculate the tf\*idf score. The reduce function does not calculate that score, but instead sums up the term frequencies on the document. This is needed because the term can be in many fields of the document. This is what the reducer looks like

{% highlight python linenos %}
reduce_function = bson.Code(
    "function (key, values) {"
    " var result = { tf: 0, df: 0};"
    " values.forEach(function(value) {"
    "   result.tf += value.tf;"
    "   result.df = value.df;"
    " });"
    " return result;"
    "}")
{% endhighlight %}

The scoring is done by a finalize function and it is straight forward

{% highlight python linenos %}
finalize_function = bson.Code(
    "function (key, value) {"
    " var score = value.tf * Math.log(total / value.df);"
    " return score;"
    "}")
{% endhighlight %}

Bringing this all together is a call to MongoDB map reduce using inline and with a query on the terms key. It also informs the finalize function of the total documents in the database using Javascript scope. The results from the Map Reduce call are then sorted by their scores and filtered to offset and limit

{% highlight python linenos %}
total = self.document_storage.count()

scope = dict(total=total)

results = self.collection.map_reduce(
    map_function,
    reduce_function,
    finalize=finalize_function,
    scope=scope,
    query={'term': { '$in': list(terms)}},
    out=bson.son.SON(dict(inline=1)))
    scored_results = sorted(
       results['results'],
       key=lambda x: x['value'],
       reverse=True)[offset:limit]
{% endhighlight %}

Running the Demo
----------------

The code is pushed to Github here: https://github.com/SupermanScott/Arya. Just checkout the repo, pip install -r requirements.txt and follow the instructions in the README. It is currently pretty bare bones, it was intended as a non-trivial exercise in MongoDB not a full on search solution.

Issues and Improvements
-----------------------

The system is currently hard coded with one tokenizer and one analyzer. This can easily be changed. The searcher returns the document and the score it received but not where the term is, or any information on how to 'highlight' the result. This is doable by adding in the required information into the match embedded document and processing it out in the Map Reduce phase. There is no query caching in this system. Paging through the results will result in duplicate work. It would be best to actually cache the output of the map reduce into Redis using a sorted set. The Redis key would have to be derived from the query.
