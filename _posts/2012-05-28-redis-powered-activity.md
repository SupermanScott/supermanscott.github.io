---
title: Redis Powered Activity
nav: blog
layout: default
summary: A fan out on write Redis powered activity with aggregation on type, user and day.
front: true
---
Every job I have ever had has asked me to build an activity feed. An activity feed, for those uninitiated is a feed of things that your friends/followers have done on a website. Sometimes it even contains things that were done by other people to your content that you don't follow/friends with. But nonetheless, it is a log of things that have happened to people or content that user cares about. I have never once been fully happy with my approach and because I have now been tasked with building this a fourth time, I have finally formed an opinion on the best way to build this.



First a little motivation on what this post will be building. The code contained in the github repository is my attempt to build an activity feed much like Instagram\'s. Instagram\'s Following feed lists things that people you are following have done on the application, aggregated by person, activity and day. So for instance, Bob liked 3 pictures on Instagram today, that would be one feed item with a title like \"Bob liked 3 photos\" and list all three photos in the item. So this post will build this functionality, an activity per user that aggregates activities by all the people that user if following by day and by type.

Solution
--------

This example code uses Celeryd, Redis for both feed storage, queue storage, cannocial store of activities and storage of the social graph. I recommend using a full disk backed store for both the cannocial store and the social graph but this is a toy solution and it is easier to get up and running by just using Redis. The presented solution is idempotent and will showcase the power of Redis sets and Redis transactions. Idempotent in this case means that queuing up the exact same same activity 1 or more times results in the database being in the exact same state. So if the activity fails somehow half way through the process, it is safe to requeue the activity and it will not produce duplicates.



There are 4 celeryd tasks, new\_activity, follow\_user, unfollow\_user and delete\_activity. New activity task will fan out the activity to each of the actor\'s follower\'s feeds and then trim those feeds to their max size. Follow user task for go through the followed user\'s activities and write them to the actor\'s feed, trimming it down to max size as well. Unfollow user and delete activity will undo both of those actions.



The aggregation is achieved by using Sets in Redis. The key to the Set is a combination of the type, user_id and day it occured. Each time an activity happens it is added to this set. The power of Redis Sets in this case allows for the code to not care if the activity is already in that Set. The same activity can be added multiple times and it does not result in duplicates. This Set is the aggregation, all three videos that Bob liked will be added to this set. If he like 2 other pictures yesterday, those 2 activities will be in another Set with a different key.



After the activity is written to its aggregation set, the key to that aggregation set is inserted or updated into the Sorted Set for each follower\'s feed. The follower\'s feed contains keys to the aggregation set and the score is the latest timestamp of all the activities in the aggregation set. Because part of the aggregation key is the day the activities happened, this will result in a nicely ordered list of recent activities. Each aggregation set also has an associated counter that is incremented and decremented when the aggregation key enters or leaves an activity feed. This allows it to garbage collect aggregatation sets that are no longer referenced by any feeds.



The code also contains a profile Sorted Set. This behaves exactly like the follower feed Sorted Set but contains aggregation keys only to the things the specific user did. This feed makes it easy to list everything the user has done sorted by the max timestamp of the aggregations.



Following a user will result in filling the user\'s feed with the followed user\'s activities. It does this by getting all the activities that followed user has done, adding to the aggregate and thus creating the aggregate if it didn\'t exist at that moment, and adding the aggregate to the user\'s feed.



The interesting parts are when writing to a Sorted Set and when trimming a Sorted Set. The code tries to enforce a size limit on the Sorted Sets of 10. This is small enough to make it easy to play with. Because of this, each aggregation has a counter associated with it. This counter indicates the number of feeds that reference this aggregation. Every time an aggregate enters or leaves a feed this counter needs to be incremented or decremented accordingly. To achieve this, the code uses the Watch command to ensure that nothing significant changes before both adding or removing from feed and incrementing or decrementing the counter. This ensures that adding and removing happen together with incrementing or decrementing the counter. All of these also helps ensure idempotent writes.



Demo
----

Installing should be easy using pip and requirements.txt so go download it: https://github.com/SupermanScott/redis-activity-example

Once done boot up the python shell to fake people doing stuff. First have user 1 do a few things

{% gist 2821439 gistfile1.py %}


Then run celeryd to have it process the tasks.

Now no one is following user 1, so when celeryd process is run, there will be 9 keys like so:

{% gist 2821453 gistfile1.txt %}


Now if you execute a follow action by having user 2 follow user 1:

{% gist 2821467 gistfile1.py %}


This should introduce two new keys in Redis: "activity\_feed:1:2" and "followers:1". The followers key does not interest us, but the activity_feed one does. This means that user 2 now has an activity feed filled with user 1\'s activities. In fact, querying that, it shows the two aggregations of likes and pictures.

{% gist 2821475 gistfile1.txt %}

Summary

This post demostrates how to use Redis Set and ZSets to build an activity feeds. It shows how to leverage the properties of Redis Sets and ZSets with the Watch command to create idempotent system that is robust to failures. It should be straight forward to take this example code and apply it to Django, Flask, Bottle or whatever python framework to build an Activity Feed system.
