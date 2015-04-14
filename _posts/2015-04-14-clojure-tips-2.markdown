---
layout: post
title:  "Clojure tips & tricks #2: Destructuring sets"
date:   2015-04-14 
categories: clojure.core
---

Destructuring presents an elegant way to deal with collections. For eg.
{% highlight clojure %}
user=> (let [[a b & d] [1 2 3 4]] (+ a b))
3
user=> (let [[a b & d] '(1 2 3 4)] d)
(3 4)
{% endhighlight %}

That tempts one to try the same with sets and maps. But alas!
{% highlight clojure %}
user=> (let [[a b c & d] {:foo 1 :doo 2}] d)
UnsupportedOperationException nth not supported on this type: PersistentArrayMap  clojure.lang.RT.nthFrom (RT.java:857)
user=> (let [[a b c & d] #{1 2 3 4}] d)
UnsupportedOperationException nth not supported on this type: PersistentHashSet  clojure.lang.RT.nthFrom (RT.java:857)
{% endhighlight %}

When you think about it, it actually makes sense, doesn't it? Vectors and Lists have a determined order of traversal and therefore it is clear how to destructure those collections to the first element, second element etc. But for maps and sets, there is no implicit ordering for the contained element and so clojure refuses to destructure them. Hold on! Sorted sets do have a clear order, namely the sort order. Surely, those should destructure well ..
{% highlight clojure %}
user=> (let [[a b c & d] (sorted-set 1 2 3 4)] d)
UnsupportedOperationException nth not supported on this type: PersistentTreeSet  clojure.lang.RT.nthFrom (RT.java:857)
{% endhighlight %}

Jeez! Why wouldn't that work? Looking closely at the [source code](https://github.com/clojure/clojure/blob/clojure-1.6.0/src/jvm/clojure/lang/PersistentTreeSet.java#L77), it's clear that a sorted set (or `PersistentTreeSet`) differs from the regular set only in one manner - its response to `seq`, where it remembers to sort the elements in ascending or descending order. Since there is no internal order to the collection, destructuring would have to first call `seq` on the sorted set and then destructure it. As a design decision, this is instead left to the user as explicit action. i.e.
{% highlight clojure %}
user=> (let [[a b c & d] (seq (sorted-set 1 2 3 4))] d)
(4)
{% endhighlight %}

Destructuring uses the `nth` function internally. `nth` works only on collections that answer the interfaces `Indexed` (vectors) or `Sequential` (lists). In the former case, `nth` works in constant time whereas for the latter, it takes linear time.
