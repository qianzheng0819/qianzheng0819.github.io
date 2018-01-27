---
layout: post
title:  "Java并发编程实践笔记"
date:   2018-1-23 10:10:00 +0800
categories: 学习
tags:   java
description:
---
1.Publication and Escape.Publishing an object means making it available to code outside of its current scope,such as by storing a reference to it where other code can find it, returning it from a nonprivate method, or passing it to a method in another class.An object that is published when it should not have been is said to have escaped;

2.Neither the Java Language Specification nor the Java Memory Model formally defines immutability, but immutability is not equivalent to simply declaring all fields of an object final.An object whose fields are all final may still be mutable, since final fields can hold references to mutable objects.
An object is immutable if:
* Its state cannot be modified after construction;
* All its fields are final;
* It is properly constructed(the this reference does not escape during construction)

3.Counter in Listing4.1 shows a typical example of this pattern.It encapsulates one state vailable,`value`, and all access to that state variable is through the methods of Counter, which are all synchronized.
Listing4.1:
{% highlight java%}
@ThreadSafe
public final class Counter {
    @GuardedBy("this")  private long value = 0;

    public synchronized long getValue() {
        return value;
    }
    public synchronized long increment() {
        if (value == Long.MAX_VALUE)
            throw new IllegalStateException("counter overflow");
        return ++value;
    }
}
{% endhighlight %}

Listing 4.3. Guarding State with a Private Lock.
{% highlight java %}
public class PrivateLock {
    private final Object myLock = new Object();
    @GuardedBy("myLock") Widget widget;

    void someMethod() {
        synchronized(myLock) {
            // Access or modify the state of widget
        }
    }
}
{% endhighlight %}

Listing 4.4. Monitor-based Vehicle Tracker Implementation
{% highlight java %}
@ThreadSafe
public class MonitorVehicleTracker {
    @GuardedBy("this")
    private final Map<String, MutablePoint> locations;

    public MonitorVehicleTracker(Map<String, MutablePoint> locations) {
        this.locations = deepCopy(locations);
    }

    public synchronized Map<String, MutablePoint> getLocations() {
        return deepCopy(locations);
    }

    public synchronized MutablePoint getLocation(String id) {
        MutablePoint loc = locations.get(id);
        return loc == null ? null : new MutablePoint(loc);
    }

    public synchronized void setLocation(String id, int x, int y) {
        MutablePoint loc = locations.get(id);
        if (loc == null)
            throw new IllegalArgumentException("No such ID: " + id);
        loc.x = x;
        loc.y = y;
    }

    private static Map<String, MutablePoint> deepCopy(Map<String, MutablePoint> m) {
        Map<String, MutablePoint> result = new HashMap<String, MutablePoint>();
        for (String id : m.keySet())
            result.put(id, new MutablePoint(m.get(id)));
        return Collections.unmodifiableMap(result);
    }
}
{% endhighlight %}

Listing 4.5. Mutable Point Class Similar to Java.awt.Point
{% highlight java %}
@NotThreadSafe
public class MutablePoint {
    public int x, y;

    public MutablePoint() {
        x = 0;
        y = 0;
    }
    public MutablePoint(MutablePoint p) {
        this.x = p.x;
        this.y = p.y;
    }
}
{% endhighlight %}

Evne though MutablePoint is not thread-safe, the tracker class is.Neither the map nor any of the mutable points it contains is ever published. When we need to a return vehicle locations to callers, the appropriate values are copied using either the MutablePoint copy constructor or deepCopy, which createa a new Map whose valuse are copies of the keys and values from the old Map.

Listing 4.7. Delegating Thread Safety to a ConcurrentHashMap
{% highlight java %}
@ThreadSafe
public class DelegatingVehicleTracker {
    private final ConcurrentMap<String, Point> locations;
    private final Map<String, Point> unmodifiableMap;

    public DelegatingVehicleTracker(Map<String, Point> points) {
        locations = new ConcurrentHashMap<String, Point>(points);
        unmodifiableMap = Collections.unmodifiableMap(locations);
    }

    public Map<String, Point> getLocations() {
        return unmodifiableMap;
    }

    public Point getLocation(String id) {
        return locations.get(id);
    }

    public void setLocation(String id, int x, int y) {
        if (locations.replace(id, new Point(x, y)) == null)
            throw new IllegalArgumentException("invalid vehicle name: " + id);
    }
}
{% endhighlight %}

4.If a class is composed of multiple independent thread-safe state variables and has no operations that have any invalid state transitions, then it can delegate thread safety to the underlying state variables.

5.ConcurrentHashMap is a hash-based Map like HashMap, but it uses an entirely different locking strategy that offers better concurrency and scalability.Instead of synchronizing every method on a common lock,restricting access to a single thread at a time,it uses a finer-grained locking mechanism called lock striping to allow a greater degree of shared access.Arbitrarily many reading threads can access the map concurrently, readers can access the map concurrently with writers, and a limited number of writers can modify the map concurrently.The result is far higher throughput under concurrent access, with little performance penalty for single-threaded access.
