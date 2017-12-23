---
title: "Aggregating data with Rx.concatMapEager"
date: 2017-12-17T15:37:01+01:00
tags:
 - rx
 - rxjava
 - concat map eager
thumbnailImagePosition: left
thumbnailImage: images/flatmap-marble.png
---

Sometimes, we have the need of aggregating data at read time.
The situation could be that we are developing in client side without proper APIs, 
or maybe our developing context is a backend service without all the needed information.

<!--more-->

Let's imagine the next scenario:
We have the [Follower](https://github.com/odin-delrio/coding-tests/blob/master/rx-concat-eager/src/main/java/org/odin/Follower.java) entity: 

```
Follower
    id
    name
```

We can obtain our followers by using our [FollowersRepository](https://github.com/odin-delrio/coding-tests/blob/master/rx-concat-eager/src/main/java/org/odin/FollowersRepository.java), but, 
we want to return to our clients something like:

```
AggregatedFollower
    id
    name
    numberOfFollowers
    numberOfFollowings
```

We will need to obtain the other fields from somewhere else... 
In this case, we have the [PublicProfile](https://github.com/odin-delrio/coding-tests/blob/master/rx-concat-eager/src/main/java/org/odin/PublicProfile.java) entity (with the corresponding repository) which have the followers/followings stats.

Solving this in an imperative way could be something like:

```
// pseudocode
aggregatedFollowers = []
followers = FollowersRepostiry.get(targetUserId)

foreach(followers as follower)
    aggregatedFollowers += new AggregatedFollower(
        follower, 
        PublicProfileRepository.get(follower.getId())
    )
    
return aggregatedFollowers
```

> ### Wait, do you want to solve this by doing N calls? why aren't you listening to service events and replicating the needed data locally?
Yes, this could not be the best option for a high traffic application, 
if your followers page contains 20 elements, performing 20 http calls in each request doesn't sound good, but:
 
> what if your API client is an internal back-office? 

> or what if you are simply want to test a feature with the 1% of your users? 
maybe you don't want to change your replication data logic at this point.


By using RxJava you can deal with this [n+1](https://secure.phabricator.com/book/phabcontrib/article/n_plus_one/) problem in a easy and parallelized way.


#### concatMap
I knew that the flatMap operator does not preserve the order of the elements... 
so, since order matters for my case, I searched a bit and I found the concatMap and I tried it:

```java
public Flowable<AggregatedFollower> getFollowers(String userId) {
  return followersRepository
    .getFollowers(userId)
    .concatMap(
        follower -> publicProfileRepository
            .getPublicProfile(follower.getId())
            .map(publicProfile -> new AggregatedFollower(follower, publicProfile))
            .toFlowable()
    );
}
```

And, it worked! But, calls are made sequentially, so, if each publicProfile call takes 1 second you will have to wait
N seconds to complete the request...

The fact that the concatMap is executed sequentially is useful if your are performing write operations. For our case,
in which we are only reading, we can assume do all the job at the same time.

#### flatMap and manual ordering
When I did this the first time, I was using RxJava 1 (RxJava 2 was not released), so, no other options existed and I
wrote [custom logic to preserve the order](https://github.com/odin-delrio/coding-tests/blob/master/rx-concat-eager/src/main/java/org/odin/aggregatedfollowers/FlatMapWithManualOrderFollowersRepository.java)...

```java
  public Flowable<AggregatedFollower> getFollowers(String userId) {
    return followersRepository
        .getFollowers(userId)
        .toList()
        .toFlowable()
        .flatMap(
            followersList -> {
              List<String> orderedIds = followersList
                  .stream()
                  .map(Follower::getId)
                  .collect(Collectors.toList());

              return Flowable
                  .fromIterable(followersList)
                  .flatMap(follower ->
                      publicProfileRepository
                          .getPublicProfile(follower.getId())
                          .map(publicProfile -> new AggregatedFollower(follower, publicProfile))
                          .toFlowable()
                  )
                  .toSortedList(Comparator.comparingInt(o -> orderedIds.indexOf(o.getFollower().getId())))
                  .toFlowable()
                  .flatMap(Flowable::fromIterable);
            }
        );
  }
```

But, don't do that! Use the RxJava 2 concatMapEager instead!

#### concatMapEager
Right now, with RxJava 2, you can just do this by using the `concatMapEager` operator ;)
```java
public Flowable<AggregatedFollower> getFollowers(String userId) {
  return followersRepository
    .getFollowers(userId)
    .concatMapEager(
        follower -> publicProfileRepository
            .getPublicProfile(follower.getId())
            .map(publicProfile -> new AggregatedFollower(follower, publicProfile))
            .toFlowable()
    );
}
```

And that's it!

#### Note about parallelism
Remember that RxJava is synchronous by default, so, this examples work in a parallelized way because I'm
configuring the schedulers in my rx chains by calling to the `subscribeOn` method when needed:

```
public class PublicProfileRepository {
  // ...
  public Maybe<PublicProfile> getPublicProfile(String userId) {
    return Maybe
        .fromCallable(() -> blockingGetPublicProfile(userId))
        .subscribeOn(Schedulers.io())
        .filter(Objects::nonNull);
  }
  // ...
```

You can see all the code in my github repository: https://github.com/odin-delrio/coding-tests/tree/master/rx-concat-eager

