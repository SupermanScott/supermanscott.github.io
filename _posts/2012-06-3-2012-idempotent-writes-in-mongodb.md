---
title: Idempotent Writes in MongoDB
nav: blog
layout: default
summary: Approach to doing idempotent writes in Mongo
front: true
---
I was first introduced to this idea of Idempotent writes just a few weeks ago at MongoSF. I attended a talk by Greg Brockman about Stripe and High Availablitity with MongoDB for Fun and Profit. There were several interesting and useful tips in his talk and its worth a read on his blog, but the one thing he really didn't mention was how Stripe achieves idempotent writes. Achieving Idempotence means that all of your writes to your database can be applied over and over again and still acheive the same results. Having let this bake a bit in my mind, I wrote my previous post on building an Activity feed. It was my first real attempt at achieving idempotent writes. In that post, Redis' Sets and Sorted Sets are used to achieve idempotent writes. With Sorted Sets and Sets adding the same member has little affect on the Set itself. It will not insert duplicates. For the rest of this post, I will present my solution for achieving Idempotence with MongoDB.

Solution
--------

The genesis of my solution is as follows: in order to ensure that the MongoDB inserts hasn't happened yet the code has to compare the contents of the document with every other document in the collection. Of course, this would mean querying on every field, most of which will not have an index on it. Luckily, there is a straight forward way to detect duplicate documents without having to query on every field. Inspired by Git, the solution uses sha1 of the json encoded representation of the document as the comparison operator. Using this and by adding an index on that field, it makes it effiecient to query every document in the collection to determine if there is a duplicate already there. This field can also be used for updates to ensure that the version of the document the code is changing is the up to date version of the document. By executing a findAndModify query with the find part specificify the sha1 field, it ensures that it only updates the expected version of the document. If it doesn't find the expected version, it throws and Exception so that the calling code can figure out what to do. For example, the calling code might read the lastest version and check to see if its intended changes were already applied. If they weren't apply them and execute the update again. If they were, just continue on and pretend it executed the write.

To achieve and demonstrate this technique, I wrote a quick extension to [MongoKit's](http://namlook.github.io/mongokit/) Document class. The magic of it is just in the overridden save method. This method now uses findAndModify only, but for inserts executes an upsert. Here is the implementation

{% highlight python linenos %}
# -*- coding: utf-8 -*-
# Copyright (c) 2012, Scott Reynolds
# All rights reserved.

from mongokit import Document, UpdateQueryError
from pymongo.errors import OperationFailure
from bson import BSON
import hashlib

class IdempotentDocument(Document):
    """
    Document that is able to ensure that any writes are
    idempotent and when the document write is not
    idempotent, throws an Exception.
    """
    structure = {
        '_versioned_id': unicode,
    }

    def save(self, validate=None, safe=True, *args, **kwargs):
        """
        save the document into the db.
        """
        old_version = ''
        if '_versioned_id' in self and self['_versioned_id']:
            old_version = self['_versioned_id']

        # Find the properties of the document that will be
        # used for the hash of the document and the properties
        # to save to the document.
        hashable_properties = {}
        updates = {}
        for key, value in self.iteritems():
            # don't update the _versioned_id. it will be set
            # later on below with the new _versioned_id. This
            # pattern allows this property to be altered yet!
            # be saved properly as the sha1 of the document.
            if key != '_id' and key != '_versioned_id':
                updates[key] = value
                hashable_properties[key] = value

        # Version id is the sha1 of the hashable contents.
        # @TODO: would be nice to make hashable properties be
        # definable on the model.
        new_version = unicode(hashlib.sha1(BSON.encode(
            hashable_properties)).hexdigest())
        self['_versioned_id'] = updates['_versioned_id'] = \
            new_version

        if validate is True or
            (validate is None and
                self.skip_validation is False):
            self.validate(auto_migrate=False)
        else:
            if self.use_autorefs:
                self._make_reference(self, self.structure)

        # Basically, if this is an update AND the old one
        # doesn't have the same sha1 as the new one.
        if old_version and old_version != new_version:
            self['_versioned_id'] = new_version
            self._process_custom_type('bson', self,
                self.structure)
            new_data = self.collection.find_and_modify(
                {'_versioned_id': old_version},
                    {'$set': updates}, new=True)
            self._process_custom_type('python', self,
                self.structure)
            if not new_data:
                raise UpdateQueryError(
                    "Document has changed since it was last"\
                        "loaded")
        # This means it is a new document not loaded from
        # the database. Try to save it and if it fails,
        # that means someone beat us too it.
        elif not old_version:
            self['_id'] = new_version
            try:
                self.collection.find_and_modify(
                    {'_id': self['_id'],
                        '_versioned_id': new_version},
                            {'$set': updates},
                                new=True, upsert=True)
            except OperationFailure:
                raise UpdateQueryError(
                    "Document has already been created")
{% endhighlight %}

Restrictions
------------

The obvious restriction with this techinque is it requires that all writes happen on the most up to date document. If the system has a bunch of small writes to seperate fields of the document it can cause a ton of contention and load on MongoDB. It is also very difficult to assess if an $incr operation was actually applied. But other advance ones like $push and $pull would be easy to assess. The techinque closely monitors the WATCH command of Redis which isn't always the best model for these types of operations.
