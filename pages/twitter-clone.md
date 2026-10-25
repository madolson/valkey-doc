VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=6 nit=7

# Patterns example (`twitter-clone`)

https://valkey.io/topics/twitter-clone/

A 2009 Redis tutorial with "Redis" swapped for "Valkey", which left anachronisms, a broken ZADD example, and an "upcoming" cluster that shipped years ago. Fix the ZADD example and the persistence claim first.

## Findings

### F1 [minor] EXAMPLE: ZADD example omits the score and fails (`topics/twitter-clone.md:434`)
**Doc says:**
> We just need to `ZADD` calls:
>
>     ZADD following:1000 5000
>     ZADD followers:5000 1000
**Problem:** ZADD needs a score before each member. Both lines error on 9.1.2. The page itself (lines 202-213) says the score is the follow time, so the example contradicts the data layout. Also "We just need to `ZADD` calls" is missing "two".
**Evidence:**
```
$ valkey-cli -p 21520 zadd following:1000 5000
ERR wrong number of arguments for 'zadd' command
```
**Fix:** "We need two `ZADD` calls, using the current Unix time as the score:" then `ZADD following:1000 1401267618 5000` and `ZADD followers:5000 1401267618 1000`.

### F2 [minor] INACCURATE: "Valkey stores data permanently" ignores the persistence config (`topics/twitter-clone.md:27`)
**Doc says:**
> it is persistent, i.e. when your application ends, the data doesn't go away.

and line 31:
> Valkey stores data permanently
**Problem:** Valkey is in-memory. Durability depends on RDB/AOF settings. With 9.1.2 defaults, AOF is off and RDB snapshots run on a schedule, so writes since the last snapshot are lost on a crash. A reader building an app with "Valkey as the only database" (line 6) needs to know this.
**Evidence:**
```
$ valkey-cli -p 21520 config get save
save
3600 1 300 100 60 10000
$ valkey-cli -p 21520 config get appendonly
appendonly
no
```
**Fix:** Line 27: "it can be persisted to disk, so the data can survive a restart (see [Persistence](persistence.md))." Line 31: "If I later ask ... Valkey replies with *bar*:". Add one sentence before Prerequisites: "If Valkey is your only database, enable AOF (`appendonly yes`) so you lose at most about one second of writes on a crash."

### F3 [minor] OUTDATED: "the upcoming Valkey Cluster" (`topics/twitter-clone.md:451`)
**Doc says:**
> you may use client-side sharding, or something like a sharding proxy like Twemproxy, or the upcoming Valkey Cluster.
**Problem:** Cluster mode has shipped in every Valkey release. Calling it "upcoming" tells readers it is not ready and points them to a third-party proxy instead.
**Evidence:** topics/cluster-tutorial.md exists and is published at https://valkey.io/topics/cluster-tutorial/ (HTTP 200).
**Fix:** "Retwis does not perform any multi-key operations, so it scales with [Valkey Cluster](cluster-tutorial.md) or with client-side sharding." Then cut line 453-454's duplicate link sentence down to "However, the point to stress is ...".

### F4 [minor] OUTDATED: History note is garbled, with a typo and an anachronism (`topics/twitter-clone.md:8`)
**Doc says:**
> Note: the original version of this article was written in 2009 when Redis OSS was
> released. It was not exactly clear at that time that the data model was
> suitable to write entire applications. There were many cases of
> applications using Valkeyt as their main store, so the goal of the article today
> is to be a tutorial for newcomers.
**Problem:** "Valkeyt" is a typo. "There were many cases" contradicts the previous sentence (it was not clear in 2009, yet there were many cases), because the original "Now after 5 years there are many cases" was lost in the rename. Valkey did not exist in 2009. The author voice ("I", line 17, 27, 444) belongs to the Redis creator and is not attributed.
**Fix:** "Note: this article was originally written in 2009 for Redis, by Salvatore Sanfilippo, the author of Retwis. Today many applications use Valkey as their main store, so the goal of the article is to be a tutorial for newcomers."

### F5 [minor] CONFUSING: ZINTERSTORE example computes common followees, not common followers (`topics/twitter-clone.md:437`)
**Doc says:**
> using `ZINTERSTORE` we can have the intersection of `following` of two different users, so ... "you and Alice have 34 followers in common"
**Problem:** Intersecting two `following:<id>` sets gives users both people follow, not shared followers. Also `ZINTER` (no destination key) exists in 9.1.2 and fits a read-only "in common" lookup better.
**Evidence:** Keys defined at line 208-209: `following:1000 => Sorted Set of uids of all the following users`.
**Fix:** "For example, `ZINTER 2 followers:1000 followers:5000` returns the followers two users have in common, so a profile page can say "you and Alice have 34 followers in common"."

### F6 [minor] INACCURATE: Build-and-run instructions give the wrong binary path (`topics/twitter-clone.md:172`)
**Doc says:**
> Just get the source, build with `make`, run with `./valkey-server`, and you're ready to go.
**Problem:** `make` at the repo root places the binary in `src/`. `./valkey-server` from the root does not exist. Readers who installed a package need neither step.
**Evidence:** The prebuilt tree used for this review has the binary at `~/docreview/valkey/src/valkey-server`.
**Fix:** "You also need a running Valkey server. See [Installation](installation.md), or build from source with `make` and run `./src/valkey-server`. No configuration is required to run Retwis locally."

### F7 [nit] INACCURATE: LTRIM 0 1000 keeps 1001 elements, not 1000 (`topics/twitter-clone.md:382`)
**Doc says:**
> This is used in order to trim the list to just 1000 elements.
**Problem:** LTRIM end index is inclusive. The code at line 371 is `$r->ltrim("timeline",0,1000);`.
**Evidence:**
```
(after 1100 LPUSH timeline ...)
$ valkey-cli -p 21520 ltrim timeline 0 1000
OK
$ valkey-cli -p 21520 llen timeline
(integer) 1001
```
**Fix:** "trim the list to the newest 1001 elements (indexes 0 to 1000, inclusive)." Also fix the code comment at line 368-369.

### F8 [nit] STYLE: Typos and grammar errors (`topics/twitter-clone.md:275`)
**Doc says:**
> These are the logical steps preformed by the `isLoggedIn` function
**Problem:** Same pattern elsewhere: line 15 "with little efforts"; line 187 "key-values stores"; line 220 "the posts of created by"; line 376 "the loop will `LPUSH` the push the post"; line 437 "Having these things as separate sets"; line 449 "multi-keys operation".
**Fix:** "performed"; "with little effort"; "key-value stores"; "the posts created by"; "the loop will `LPUSH` the post"; "multi-key operations".

### F9 [nit] CONFUSING: Prose names a function that does not exist (`topics/twitter-clone.md:422`)
**Doc says:**
> `showUserPosts` gets a range of posts and then passes them to `showPosts`.
**Problem:** The function in the code block is `showPost` (line 394).
**Fix:** "passes them to `showPost`."

### F10 [nit] BROKEN: Stray asterisk renders literally (`topics/twitter-clone.md:427`)
**Doc says:**
> deep pagination of million of items, it is better to resort to Sorted Sets
> instead.*
**Problem:** No opening `*`, so the live page shows "instead.*".
**Evidence:** `curl -sL https://valkey.io/topics/twitter-clone/ | grep -o 'instead\.[^<]\{0,10\}'` returns `instead.*`.
**Fix:** Remove the trailing `*`. Also "million of items" to "millions of items", and "Lists are backed by linked lists" to "Lists are backed by a linked list of listpacks, so `LRANGE` is O(S+N) where S is the start offset".

### F11 [nit] STYLE: Unprofessional and gendered phrasing (`topics/twitter-clone.md:444`)
**Doc says:**
> and this one was monkey ass slow... Imagine the results with more recent hardware.
**Problem:** Profanity in official docs. Also "her/his own posts" (line 219) and "following herself/himself" (line 348). "Gentle reader, if you read till this point you are already a hero. Thank you." (line 444) is filler.
**Fix:** Cut "Gentle reader ... Thank you." Replace the last sentence with "and that server was very slow." Use "their own posts" and "following themselves".

### F12 [nit] OUTDATED: Data type list is incomplete (`topics/twitter-clone.md:67`)
**Doc says:**
> Valkey supports Lists, Sets, Hashes, Sorted Sets, Bitmaps, and HyperLogLog types as values
**Problem:** Omits Streams and geospatial indexes, both in 9.1.2. Readers looking for the full list get a partial one.
**Fix:** "Valkey supports Lists, Sets, Hashes, Sorted Sets, Streams, Bitmaps, HyperLogLogs, and geospatial indexes (see [Data types](data-types.md))."

### F13 [nit] AI-NATIVE: Title says nothing about the page (`topics/twitter-clone.md:2`)
**Doc says:**
> title: "Patterns example"
**Problem:** The title does not name the subject (a Twitter clone data model). The first paragraph also argues a 2009 point ("The programming community has traditionally considered key-value stores...") before saying what the reader will build.
**Fix:** `title: "Tutorial: build a Twitter clone with Valkey"`. Open with "This tutorial walks through the data model of Retwis, a simple Twitter clone in PHP that uses Valkey as its only database. It covers ID generation with INCR, objects as hashes, follower graphs as sorted sets, timelines as lists, and cookie authentication."
