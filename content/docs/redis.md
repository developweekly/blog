---
title:  "Redis"
date:   2017-01-27 00:44:07 +0200
categories: database redis
---

# Introduction to Redis

**Jan 27, 2017**<!-- \ -->
<!-- <sup>Last modified: **Dec 2, 2018**</sup> -->

It's been a long break for the blog posts. But this means I have been working my ass off, so there are tons of stuff piled up to talk about! First one coming to my mind is <a href="http://redis.io" target="_blank">Redis</a>.

Redis is an in-memory data structure store. For this reason, it's incredibly fast. It has key-value structure and supports data structures such as <a href="http://redis.io/topics/data-types-intro#strings">strings</a>, <a href="http://redis.io/topics/data-types-intro#hashes">hashes</a>, <a href="http://redis.io/topics/data-types-intro#lists">lists</a>, <a href="http://redis.io/topics/data-types-intro#sets">sets</a>, <a href="http://redis.io/topics/data-types-intro#sorted-sets">sorted sets</a> and so on.

It can be used either as a database, if relational ones keeps bugging you. Or you can cache your data in Redis, because it already stores everything in memory. Plus it has the capability of a message broker, where you can publish messages through Redis, or might even use it as your load balancer and task queueing purposes.

It has a great <a href="http://try.redis.io/">interactive tutorial</a> where you can jump into it and start trying Redis immediately.

# Connecting to Redis Server

After you follow the instructions and set up the redis-server, import the module "redis" in Python and create a connection by using StrictRedis as:

```python
con = redis.StrictRedis(host="localhost", port=6379, db=0)
```

You can either check your Redis db in terminal by redis-cli command, or you can download one of the desktop managers for Redis. I have been using <a href="https://redisdesktop.com" target="_blank">Redis Desktop Manager</a> and it was free back then. You can still find its earlier versions and keep using it as free. There is also <a href="http://fastoredis.com">FastoRedis</a> and many other open source GUI for Redis.

# Basic Commands

Once you get your redis-server up and running, and you've got yourself a connection, why keep waiting? Here are some basic Redis keys commands:

```python
con.set("key", "value")

con.get("key")
>> "value"

con.del("key")
```

These are just about enough for your key-value storage purposes. For storing lists in Redis:
<ul>
	<li>To push an item to the tail of a list:</li>
</ul>

```python
con.rpush("key", "item")
```

<ul>
	<li>To get the length of a list:</li>
</ul>

```python
con.llen("key")
>> 1
```

<ul>
	<li>To insert an item to the head of a list:</li>
</ul>

```python
con.lpush("key", "otheritem")
```

<ul>
	<li>To get the list, i.e. for looping throug it (from head to tail, that's what 0 & -1 stand for):</li>
</ul>

```python
con.lrange("key", 0, -1)
>> ["otheritem", "item"]
```

That's just about it! There are other commands for sets, managing queue lists, hashes, publish/subscribe messaging and so on in <a href="http://redis.io/commands" target="_blank">the command reference</a>.
