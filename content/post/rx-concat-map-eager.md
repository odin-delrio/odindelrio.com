---
title: "Aggregating data with Rx.concatMapEager"
date: 2017-12-17T15:37:01+01:00
tags:
 - rx
 - rxjava
 - concat map eager
thumbnailImagePosition: left
---

Sometimes, we have the need of aggregating data at read time.
The situation could be that we are developing in client side without proper APIs, 
or maybe our developing context is a backend service without all the needed information.

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

We will need to obtain the rest of the data from somewhere else... 
Then, we will create the [PublicProfile](https://github.com/odin-delrio/coding-tests/blob/master/rx-concat-eager/src/main/java/org/odin/PublicProfile.java) entity which have the followers/followings stats.

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
Yes, this could not be the best option for a production traffic application, 
if your followers page contains 20 elements, performing 20 http calls doesn't sound good, but:
 
> what if your API client is an internal back-office? 

> or what if you are simply want to test a feature with the 1% of your users? 
maybe you don't want to change your replication data logic at this point.


By using RxJava you can deal with this [n+1](https://secure.phabricator.com/book/phabcontrib/article/n_plus_one/) problem in a paralleled way.


#### flatMap
My first option for doing this was using the flatMap operator, assuming we have needed repositories returning
reactive types, we can do something like:

```java
public Flowable<AggregatedFollower> getFollowers(String userId) {
  return followersRepository
    .getFollowers(userId)
    .flatMap(
        follower -> publicProfileRepository
            .getPublicProfile(follower.getId())
            .map(publicProfile -> new AggregatedFollower(follower, publicProfile))
            .toFlowable()
    );
}
```

#### concatMap

But, the flatMap does not preserve the order of the elements... so, if that matter to us, we can't use the flatMap operator here.
So, after searching a bit, I thought that the concatMap operator will do the work...

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

This worked, but, calls are made sequentially, so, if each publicProfile call takes 1 second... you know.

#### concatMapEager

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
