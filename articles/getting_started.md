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

This guide covers Spyglass 1.1.x.


## Spyglass Overview

Spyglass is an idiomatic Clojure client for [Memcached](http://memcached.org/). It is simple and easy to use, strives to support
every Memcached protocol feature, both text and binary protocols as well as offer asynchronous and synchronous APIs for
most operations.

Spyglass gets its name from and is based on on Spy Memcached, a mature, heavily tested and very fast Java client for Memcached.
Spyglass' overhead is very very low.

You can use Spyglass with servers that use the Memcached protocol, for example, [Membase Server](http://www.couchbase.com/membase) or [Kestrel messaging queue](https://github.com/robey/kestrel/blob/master/docs/guide.md).


## Supported Clojure versions

Spyglass is built from the ground up for Clojure 1.3 and later.


## Supported Memcached versions

Spyglass supports Memcached 1.4.x (1.4.13 or later are recommended). Some features (like binary protocol support) may require specific point versions.
See [the list of Memcached 1.4.x release notes](http://code.google.com/p/memcached/wiki/ReleaseNotes) to learn more.


## Adding Spyglass Dependency To Your Project

### With Leiningen

    [clojurewerkz/spyglass "1.1.0"]

### With Maven

    <dependency>
      <groupId>clojurewerkz</groupId>
      <artifactId>spyglass</artifactId>
      <version>1.1.0</version>
    </dependency>

It is recommended to stay up-to-date with new versions. New releases and important changes are announced [@ClojureWerkz](http://twitter.com/ClojureWerkz).



## Connecting to Memcached

Spyglass has a single namespace: `clojurewerkz.spyglass.client`. Require it and use the `clojurewerkz.spyglass.client/text-connection`
function that take a server list string:

{% gist 6911e5c9ee201c47843b %}

`clojurewerkz.spyglass.client/bin-connection` works the same way but uses binary Memcached protocol:

{% gist ed6bb2a42b40e95884ca %}


## Setting keys

To set a key, use the `clojurewerkz.spyglass.client/set` function that takes a client instance, a string key, an expiration value (as an integer) and a value
to store and returns a future:

{% gist 79d271fa55880a21669f %}

### Expiration values

The actual expiration value sent may either be Unix time (number of seconds since January 1, 1970, as a 32-bit value), or a number of seconds starting from
current time. In the latter case, this number of seconds may not exceed `60*60*24*30` (number of seconds in 30 days); if the number sent by a client
is larger than that, the server will consider it to be real Unix time value rather than an offset from current time.

### Transcoders

TBD


## Getting keys

### Synchronous get

To get a value synchronously, use the `clojurewerkz.spyglass.client/get` function that takes a client instance, a string key, and returns a stored value:

{% gist 64d87ac77b93e9a013b7 %}

### Asynchronous get

Spyglass also provides a way to fetch a value asynchronously using the `clojurewerkz.spyglass.client/async-get` function that the same arguments but returns a future:

{% gist ade1b1d1a5781d027412 %}


## Getting multiple keys in a single operation

### Synchronous multi-get

It is possible to get multiple values in a single request using the `clojurewerkz.spyglass.client/get-multi` function that takes a client instance, a collection of keys, and returns
an immutable map of stored values:

{% gist 218b26c955e68e411f1d %}


### Asynchronous multi-get

Asynchronous multi-get is available via the `clojurewerkz.spyglass.client/async-get-multi` function that returns a future that, when dereferenced, returns a map
of results:

{% gist 0610038900a52b02a154 %}


## Deleting keys

To delete a key, use the `clojurewerkz.spyglass.client/delete` function that has the same signature as (2-arity) `clojurewerkz.spyglass.client/get`:

{% gist 3807c05c44bbbd1d21bd %}

Delete operations are always asynchronous and always return a future.


## Touching (updating expiration time) keys

To touch a key means to reset or update its expiration time using the `clojurewerkz.spyglass.client/touch` function:

{% gist cc701c1a08235ab6115d %}

When touching keys, values lower than `60*60*24*30` are treated as relative offset (a number of seconds starting from current time) and values greater than that
are treated as absolute Unix timestamps. 


## add operation

Sometimes you only need to add a value to the cache but only if does not already exist. This is what the `clojurewerkz.spyglass.client/add` function does
in a single request:

{% gist ec5949989148630aa4fe %}

This function returns a future that, when dereferenced, returns a boolean (false if the mutation did not occur, true if it did).


## replace operation

`clojurewerkz.spyglass.client/replace` is similar to `clojurewerkz.spyglass.client/add` but will replace a value with the given one if
there is already a value for the given key:

{% gist f6f6d318caf9f2fa33ec %}

This function returns a future that, when dereferenced, returns a boolean (false if the mutation did not occur, true if it did).


## gets and CAS (compare-and-swap) operations

TBD


## increments, decrements operations

TBD


## Getting stats

TBD


## Disconnecting from Memcached

To disconnect, use the `clojurewerkz.spyglass.client/shutdown` function:

{% gist bebff41cb472cafb7d8c %}

It is also possible to let all running asynchronous operations to finish by disconnecting with a timeout:

{% gist 973fb66b88e07ce2a454 %}



## Wrapping up

Congratulations, you now can use Spyglass to work with Memcached or other servers that use (e.g. [Membase Server](http://www.couchbase.com/membase) or [Kestrel messaging queue](https://github.com/robey/kestrel/blob/master/docs/guide.md)). Now you know enough to start building
a real application.

We hope you find Spyglass reliable, consistent and easy to use. In case you need help, please ask on the [mailing list](https://groups.google.com/forum/#!forum/clojure-memcached)
and follow us on Twitter [@ClojureWerkz](http://twitter.com/ClojureWerkz).


## Tell Us What You Think!

Please take a moment to tell us what you think about this guide on Twitter or the [Spyglass mailing list](https://groups.google.com/forum/#!forum/clojure-memcached)

Let us know what was unclear or what has not been covered. Maybe you do not like the guide style or grammar or discover spelling mistakes. Reader feedback is key to making the documentation better.
