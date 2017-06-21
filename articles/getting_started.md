---
title: "Getting Started with Spyglass, a Clojure client for Memcached"
layout: article
---

## About this guide

This guide combines an overview of Spyglass with a quick tutorial that helps you to get started with it.
It should take about 15 minutes to read and study the provided code examples. This guide covers:

 * Features of Spyglass
 * Clojure and Memcached Server version requirements
 * How to add Spyglass dependency to your project
 * A very brief introduction to Spyglass capabilities
 * Examples of the most common Memcached operations with Spyglass

This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/3.0/">Creative Commons Attribution 3.0 Unported License</a> (including images & stylesheets). The source is available [on Github](https://github.com/clojurewerkz/spyglass.docs).


## What version of Spyglass does this guide cover?

This guide covers Spyglass 1.2.x.


## Spyglass Overview

Spyglass is an idiomatic Clojure client for [Memcached](http://memcached.org/). It is simple and easy to use, strives to support
every Memcached protocol feature, both text and binary protocols as well as offer asynchronous and synchronous APIs for
most operations.

Spyglass gets its name from and is based on on Spy Memcached, a mature, heavily tested and very fast Java client for Memcached.
Spyglass' overhead is very very low.

You can use Spyglass with servers that use the Memcached protocol, for example, [Membase Server](http://www.couchbase.com/membase) or [Kestrel messaging queue](https://github.com/robey/kestrel/blob/master/docs/guide.md).


## Supported Clojure versions

Spyglass requires Clojure 1.7+.


## Supported Memcached versions

Spyglass supports Memcached 1.4.x (1.4.13 or later are recommended). Some features (like binary protocol support) may require specific point versions.
See [the list of Memcached 1.4.x release notes](http://code.google.com/p/memcached/wiki/ReleaseNotes) to learn more.


## Adding Spyglass Dependency To Your Project

### With Leiningen

    [clojurewerkz/spyglass "1.2.0"]

### With Maven

    <dependency>
      <groupId>clojurewerkz</groupId>
      <artifactId>spyglass</artifactId>
      <version>1.2.0</version>
    </dependency>

It is recommended to stay up-to-date with new versions. New releases and important changes are announced [@ClojureWerkz](http://twitter.com/ClojureWerkz).



## Connecting to Memcached

Spyglass has a single namespace: `clojurewerkz.spyglass.client`. Require it and use the `clojurewerkz.spyglass.client/text-connection`
function that take a server list string:

```clojure
(ns clojurewerkz.spyglass.examples
  (:require [clojurewerkz.spyglass.client :as c]))

(defn -main
  [& args]
  ;; connects to 2 Memcached servers using text protocol
  (let [tmc-client (c/text-connection "server1.internal:11211 server2.internal:11211")]
    tmc-client))
```


`clojurewerkz.spyglass.client/bin-connection` works the same way but uses binary Memcached protocol:

```clojure
(ns clojurewerkz.spyglass.examples
  (:require [clojurewerkz.spyglass.client :as c]))

(defn -main
  [& args]
  ;; connects to 2 Memcached servers using binary protocol
  (let [bmc-client (c/bin-connection "server1.internal:11211 server2.internal:11211")]
    bmc-client))
```


## Setting keys

To set a key, use the `clojurewerkz.spyglass.client/set` function that takes a client instance, a string key, an expiration value (as an integer) and a value
to store and returns a future:

```clojure
(ns clojurewerkz.spyglass.examples
  (:require [clojurewerkz.spyglass.client :as c]))

(defn -main
  [& args]
  (let [tmc (c/text-connection "127.0.0.1:11211")
        ;; set a key that expires in 5 minutes (300 seconds)
        res (c/set tmc "a-key" 300 "a value")]
    ;; results of async operations in Spyglass can be @dereferenced
    @res))
```

### Expiration values

The actual expiration value sent may either be Unix time (number of seconds since January 1, 1970, as a 32-bit value), or a number of seconds starting from
current time. In the latter case, this number of seconds may not exceed `60*60*24*30` (number of seconds in 30 days); if the number sent by a client
is larger than that, the server will consider it to be real Unix time value rather than an offset from current time.

### Transcoders

Transcoder converts between byte arrays and objects for storage in the cache. Spyglass has 4 builtin factory-functions to initialize the right transcoder object for your use-case:

 * **integer** - transcoder that serializes and unserializes only Integers.
 *ps: setting a new value requires explicit type conversion, (int clojure-number)*

 * **long**     - transcoder that serializes and unserializes only Longs.

 * **whalin**   - transcoder that provides compatibility with Greg Whalin's memcached client. It can handle most of Clojure primitives and datastructures, except nil, record, exception. It useful when you want to integrate with [gwhalin/Memcached-Java-Client](https://github.com/gwhalin/Memcached-Java-Client).

 * **serializing** - (*default*), Transcoder that serializes and compresses Java objects. It can handle most of Clojure primitives and datastructures, except nil, record, exception.

###### Usage

```clojure
(require '[clojurewerkz.spyglass.client :as c])
(require '[clojurewerkz.spyglass.transcoders :as t] :reload)
(def tc (c/text-connection "192.168.99.100:11211"))

(def nippler (t/make-transcoder :long))
(c/set tc "abc" 30 (long 123) nippler)
(c/get tc "abc" nippler)
```

###### Extending transcoders

There maybe use-cases when you want to use transcoders that can handle wider subset of Clojure primitives - like records. Or you want to compress or encrypt a content before sending a data out to a memcached server.

There're 2 very good Clojure libraries just for that: [Nippy](https://github.com/ptaoussanis/nippy) or  [Data.Fressian](https://github.com/clojure/data.fressian)

Here's a little example how to write a special purpose transcoder by extending [Transcoder](http://www.couchbase.com/autodocs/spymemcached-2.8.4/net/spy/memcached/transcoders/Transcoder.html) interface.

```clojure
(require '[taoensso.nippy :as nippy])
(import '[net.spy.memcached CachedData]
        '[net.spy.memcached.transcoders Transcoder]))

;;experimental Nippy transcoder
(deftype NippyTranscoder []
  net.spy.memcached.transcoders.Transcoder
  (asyncDecode [this dt] false) ;tells should it be encoded async
  (getMaxSize [this]
    (int CachedData/MAX_SIZE))

  (decode [this dt]
    (-> dt (.getData) nippy/thaw))

  (encode [this clj-obj]
    (CachedData. (int 0) ;optional flags for decoding
                 (nippy/freeze clj-obj)
                 CachedData/MAX_SIZE)))
```

Here's a full-list of data-types Nippy transcoder supports: [https://gist.github.com/timgluz/b89ad29316d4b2633de1](https://gist.github.com/timgluz/b89ad29316d4b2633de1)

## Getting keys

### Synchronous get

To get a value synchronously, use the `clojurewerkz.spyglass.client/get` function that takes a client instance, a string key, and returns a stored value:

```clojure
(ns clojurewerkz.spyglass.examples
  (:require [clojurewerkz.spyglass.client :as c]))

(defn -main
  [& args]
  (let [tmc (c/text-connection "127.0.0.1:11211")
        ;; set a key that expires in 5 minutes (300 seconds)
        _   (c/set tmc "a-key" 300 "a value")
        ;; fetch it back
        val (c/get tmc "a-key")]
    ;; clojurewerkz.spyglass.client/get returns a value synchronously, no need to deref
    val))
```

### Asynchronous get

Spyglass also provides a way to fetch a value asynchronously using
the `clojurewerkz.spyglass.client/async-get` function that the same arguments but returns a future:


```clojure
(ns clojurewerkz.spyglass.examples
  (:require [clojurewerkz.spyglass.client :as c]))

(defn -main
  [& args]
  (let [tmc (c/text-connection "127.0.0.1:11211")
        _   (c/set tmc "a-key" 300 "a value")
        ;; fetch the value asynchronously
        async-val (c/async-get tmc "a-key")]
    ;; deref the result (a future)
    @async-val))
```

## Getting multiple keys in a single operation

### Synchronous multi-get

It is possible to get multiple values in a single request using the `clojurewerkz.spyglass.client/get-multi` function that takes a client instance, a collection of keys, and returns
an immutable map of stored values:

```clojure
(ns clojurewerkz.spyglass.examples
  (:require [clojurewerkz.spyglass.client :as c]))

(defn -main
  [& args]
  (let [tmc (c/text-connection "127.0.0.1:11211")
        _   (c/set tmc "a-key" 300 "a value")
        _   (c/set tmc "another-key" 300 "another value")
        ;; fetch both values back in a single request
        m   (c/get-multi tmc ["a-key" "another-key"])]
    ;; clojurewerkz.spyglass.client/get-multi returns an immutable map synchronously, no need to deref
    (println m)))
```

### Asynchronous multi-get

Asynchronous multi-get is available via the `clojurewerkz.spyglass.client/async-get-multi` function that returns a future that, when dereferenced, returns a map
of results:

```clojure
(ns clojurewerkz.spyglass.examples
  (:require [clojurewerkz.spyglass.client :as c]))

(defn -main
  [& args]
  (let [tmc (c/text-connection "127.0.0.1:11211")
        _   (c/set tmc "a-key" 300 "a value")
        _   (c/set tmc "another-key" 300 "another value")
        ;; fetch both values back in a single request
        async-val (c/async-get-multi tmc ["a-key" "another-key"])]
    ;; clojurewerkz.spyglass.client/async-get-multi returns a future that, when dereferenced,
    ;; returns an immutable map
    (println @async-val)))
```

## Deleting keys

To delete a key, use the `clojurewerkz.spyglass.client/delete` function that has the same signature as (2-arity) `clojurewerkz.spyglass.client/get`:

```clojure
(ns clojurewerkz.spyglass.examples
  (:require [clojurewerkz.spyglass.client :as c]))

(defn -main
  [& args]
  (let [tmc (c/text-connection "127.0.0.1:11211")
        _   (c/set tmc "a-key" 300 "a value")
        async-val (c/delete tmc "a-key")]
    ;; deletes are always asynchronous so to get the returned value
    ;; you need to @dereference it
    @async-val))
```

Delete operations are always asynchronous and always return a future.


## Touching (updating expiration time) keys

To touch a key means to reset or update its expiration time using the `clojurewerkz.spyglass.client/touch` function:

```clojure
(ns clojurewerkz.spyglass.examples
  (:require [clojurewerkz.spyglass.client :as c]))

(defn -main
  [& args]
  (let [tmc (c/text-connection "127.0.0.1:11211")
        _   (c/set tmc "a-key" 300 "a value")
        ;; touch the given key to reset its expiration time
        async-val (c/touch tmc "a-key" 450)]
    @async-val))
```

When touching keys, values lower than `60*60*24*30` are treated as relative offset (a number of seconds starting from current time) and values greater than that
are treated as absolute Unix timestamps.


## add operation

Sometimes you only need to add a value to the cache but only if does not already exist. This is what the `clojurewerkz.spyglass.client/add` function does
in a single request:

```clojure
(ns clojurewerkz.spyglass.examples
  (:require [clojurewerkz.spyglass.client :as c]))

(defn -main
  [& args]
  (let [tmc (c/text-connection "127.0.0.1:11211")
        ;; adds a value only if does not already exist
        async-val (c/add tmc "a-key" 300 "a value")]
    (println @async-val)))
```

This function returns a future that, when dereferenced, returns a boolean (false if the mutation did not occur, true if it did).


## replace operation

`clojurewerkz.spyglass.client/replace` is similar to `clojurewerkz.spyglass.client/add` but will replace a value with the given one if
there is already a value for the given key:

```clojure
(ns clojurewerkz.spyglass.examples
  (:require [clojurewerkz.spyglass.client :as c]))

(defn -main
  [& args]
  (let [tmc (c/text-connection "127.0.0.1:11211")
        ;; replaces a value if it is already set
        async-val1 (c/replace tmc "a-new-key" 300 "a value")
        _          (c/set tmc "a-new-key" 300 "a value")
        async-val2 (c/replace tmc "a-new-key" 300 "another value")]
    (println @async-val1)
    (println @async-val2)))
```

This function returns a future that, when dereferenced, returns a boolean (false if the mutation did not occur, true if it did).


## gets and CAS (compare-and-swap) operations

TBD


## increments, decrements operations


Increment the given key by the given amount. 
Returns the new value, or -1 if we were unable to decrement or add.

```clojure
(c/incr tc "the-session-counter" 1)
```

Increment the given counter, returning the new value. A default value is used only if it doesnt exist;

```clojure
(c/incr client the-key increment-by default-value)
(c/incr tc "the-session-counter" 1 0)

(c/incr client the-key increment-by default-value time-to-live-seconds)
(c/incr tc "the-session-counter" 1 0 300)
```

Decrement follows similar interface:

```clojure
(c/decr client the-key increment-by)
(c/decr client "candies-left" 1)

(c/decr client the-key increment-by default-value)
(c/decr client "candies-left" 1 99)

(c/decr client the-key increment-by default-value time-to-live-seconds)
(c/decr client "candies-left" 1 99 300)

```


## Getting stats

Get all of the stats from all of the connections. 
Returns hash-map, in which hosts are keys and statistics are values.

```clojure
(c/get-stats tc)
(c/get-stats tc "items")
```

Accepted arguments for 2nd argument: [source](http://lzone.de/articles/memcached.htm)

* default, shows a current traffic & usage statistics (*uptime, total-items, total-connections, threads, ...*)
* **slabs** - memory metrics (*chunk-size, chunks-per-page, total-malloced*)
* **items** - items statistics (*items:14:evicted" "0", "items:8:age" "241088", "items:1:crawler_items_checked*)
* **sizes** - returns list of keys used and their statistics
* **reset** - resets statistics

## Disconnecting from Memcached

To disconnect, use the `clojurewerkz.spyglass.client/shutdown` function:

```clojure
(ns clojurewerkz.spyglass.examples
  (:require [clojurewerkz.spyglass.client :as c]))

(defn -main
  [& args]
  (let [tmc (c/text-connection "127.0.0.1:11211")]
    ;; disconnect immediately
    (c/shutdown tmc)))
```

It is also possible to let all running asynchronous operations to finish by disconnecting with a timeout:

```clojure
(ns clojurewerkz.spyglass.examples
  (:require [clojurewerkz.spyglass.client :as c]))

(defn -main
  [& args]
  (let [tmc (c/text-connection "127.0.0.1:11211")]
    ;; disconnect in 3 seconds
    (c/shutdown tmc 3 java.util.concurrent.TimeUnit/SECONDS)))
```

## Wrapping up

Congratulations, you now can use Spyglass to work with Memcached or other servers that use (e.g. [Membase Server](http://www.couchbase.com/membase) or [Kestrel messaging queue](https://github.com/robey/kestrel/blob/master/docs/guide.md)). Now you know enough to start building
a real application.

We hope you find Spyglass reliable, consistent and easy to use. In case you need help, please ask on the [mailing list](https://groups.google.com/forum/#!forum/clojure-memcached)
and follow us on Twitter [@ClojureWerkz](http://twitter.com/ClojureWerkz).


## Tell Us What You Think!

Please take a moment to tell us what you think about this guide on Twitter or the [Spyglass mailing list](https://groups.google.com/forum/#!forum/clojure-memcached)

Let us know what was unclear or what has not been covered. Maybe you do not like the guide style or grammar or discover spelling mistakes. Reader feedback is key to making the documentation better.
