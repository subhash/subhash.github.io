---
layout: post
title:  "Collections and Sequences"
date:   2015-01-02
categories: clojure.core
---

Clojure collections are literally a handful and distinguishing between the various types is tricky. Let's take a shot at it by studying the various collection functions:

**`coll?`** tells us which ones are valid collections. Expectedly, scalars like numbers and keywords are not collections. Strings are also not collections, but they can be easily converted to become collections
{% highlight clojure %}
user=> (map coll? [ '(:foo) [:foo] #{:foo} {:foo 1} :foo 1 "foo" ])
; (true true true true false false false)
{% endhighlight %}

**`counted?`** validates those collections that possibly be counted. Understandably, the ones that cannot be counted are the lazy sequences which may be infinite. Interestingly, sequence functions like `map`, `filter` etc. return lazy sequences which cannot be counted.

{% highlight clojure %}
user=> (map counted? [ [:foo] '(:foo) {:foo 1} #{:foo} (range) (map identity []) ])
; (true true true true false false)
{% endhighlight %}

**`associative?`** prefers those collections which are mapped in the form of `key:value` - namely, maps and vectors. In vectors, the mapping is from the element's index to itself. Sets are a trivial mapping from the element to itself and so are not considered associative. Lists and (lazy) sequences are accessed linearly and therefore, the concept of an index does not apply to them

{% highlight clojure %}
user=> (map associative? [[:foo] '(:foo) {:foo 1} #{:foo} (range) ])
; (true false true false false)
{% endhighlight %}

**`sequential?`** collections are those that can possibly be accessed in a linear order. Maps and sets don't quality because they have no defined order of access. 
{% highlight clojure %}
user=> (map sequential? [[:foo] '(:foo) {:foo 1} #{:foo} (range) ])
; (true true false false true)
{% endhighlight %}

A collection is simply a bag of values. It is useful to be able to deal with the different kinds of collections from a common abstraction. Towards this, Clojure defines a "list-like" abstraction called `sequence`. This is similar to lists in that it is accessed linearly using `first` and `rest`. It is possible to create a `sequence` from any `collection` by calling `seq` on it. Such a sequence would always be sequential and non-associative. In fact, most collection APIs like `map`, `filter`, `take` etc call `seq` on the parameter before working on them. This is why all these APIs work consistently on all collections.
{% highlight clojure %}
user=> (map (comp sequential? seq) [[:foo] '(:foo) {:foo 1} #{:foo} (range) ])
; (true true true true true)
user=> (map (comp associative? seq) [[:foo] '(:foo) {:foo 1} #{:foo} (range) ])
; (false false false false false)
{% endhighlight %}

`seq` also works on strings and other non-collections like Java arrays etc. The useful thing about `seq` is that it returns `nil` for empty collections and `nil`. Therefore, we have something like:
{% highlight clojure %}
user=> (let [a []]
         (if (seq a)
           (first a)
           :nothing-to-do))
:nothing-to-do
{% endhighlight %}

When `seq` is called on a collection, we get a specific implementation of sequence that presents a "list-view" of the collection. Since `list` is already a concrete implementation of sequence, it returns itself

{% highlight clojure %}
user=> (doseq [c [[:foo] '(:foo) {:foo 1} #{:foo} (range) ]] (println (class c) "->" (class (seq c))))
clojure.lang.PersistentVector -> clojure.lang.PersistentVector$ChunkedSeq
clojure.lang.PersistentList -> clojure.lang.PersistentList
clojure.lang.PersistentArrayMap -> clojure.lang.PersistentArrayMap$Seq
clojure.lang.PersistentHashSet -> clojure.lang.APersistentMap$KeySeq
clojure.lang.LazySeq -> clojure.lang.ChunkedCons
nil
{% endhighlight %}
    
