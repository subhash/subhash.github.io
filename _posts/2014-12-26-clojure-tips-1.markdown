---
layout: post
title:  "Clojure tips & tricks #1: contains?"
date:   2014-12-26 
categories: clojure.core
---

You want to check if a set contains a certain element? Easy peasy ...
{% highlight clojure %}
user=> (contains? #{:foo :bar} :foo)
false
{% endhighlight %}

How about a map?
```clojure
user=> (contains? {:foo 1 :bar 2} :foo)
true
```
Should work for vectors too, right?
```clojure
user=> (contains? [:foo :bar] :foo)
false
```
Nope. That does not work because `contains?` checks for the presence of keys, not values. And when you think about it, the keys for a vector are its indices. This is because a vector is strictly ordered and the key `1` should correspond to the second element, if any.
```clojure
(contains? [:foo :bar] 1)
true
```
This allows the `contains?` function to perform in constant or logarithmic time by searching only within the key indices.

Clojure provides a different function to check for values in a collection.
```clojure
user=> (some #(= % :foo) [:foo :bar])
true
```
Or even better:
```clojure
user=> (some #{:foo} [:foo :bar])  ; a set acts as a predicate 
:foo ; a truthy value
```

That might give you yet another idea
```clojure
user=> ((set [:foo :bar]) :foo)
:foo
user=> ((set [:foo :bar]) :doo)
nil
```    
