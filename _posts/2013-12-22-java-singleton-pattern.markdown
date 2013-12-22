---
layout: post
title:  "The Java Singleton Pattern"
date:   2013-12-22 12:00:00
tags:
 - java
 - programming
---

Much has been written about how to use singletons in Java. However, I've discovered
a neat quirk of the syntax that I feel makes your intent clearer. It's rather simple.
Here's a code example of using the initialization-on-demand holder pattern in Java. This
uses all the best practices for instantiation and safety.

    public final class SomeSingleton {
        private SomeSingleton() {}
        private static interface Holder {
            static final SomeSingleton instance = new SomeSingleton();
        }
        public static SomeSingleton getInstance() {
            return Holder.instance;
        }
        // ...
    }

The main things to take away from this are:

1. Singleton classes should be `final`! There's no point in extending a singleton, so
   should enforce that by making the class final.
2. Singletons should have a `private` constructor, and only one at that.
3. Singletons should have a `private` `static` inner class used to hold the singleton
   instance. In this case, we can use an `interface` since all it contains is a
   static field. This feels like an [obscure part of the Java language specification][jls_93],
   but it's a legit feature. If we were using a `class` instead of an `interface`, we
   could make `instance` be `private` as well, though I don't see any point.
4. Singletons should have a `public` `static` method for getting the singleton instance.
   Generically, we can call this `getInstance`, although sometimes it's named after the
   class itself like `getSomeSingleton`.
5. If constructing the singleton can fail (which can't happen in the above example), this
   pattern won't work. Back to locking everything!

So why go through all this trouble when you could just store a static instance of the
class in itself rather than an inner class? Because this singleton pattern is thread-safe
according to the [specs on JVM execution][jls_12]. This is mainly to maintain backwards
compatibility with older JVMs, but it's a relatively simple design pattern to follow.
Check out the [Wikipedia article][wiki] and the [technical details][dcl] behind the
idea. Also, this avoids synchronizing the `getInstance` method thanks to the Java
memory model.

[jls_93]: http://docs.oracle.com/javase/specs/jls/se7/html/jls-9.html#jls-9.3
[jls_12]: http://docs.oracle.com/javase/specs/jls/se7/html/jls-12.html
[wiki]: https://en.wikipedia.org/wiki/Initialization-on-demand_holder_idiom
[dcl]: http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html
