---
layout: post
title:  "Clojure tips & tricks #1: contains?"
date:   2014-12-26 
categories: clojure.core
---

You want to check if a set contains a certain element? Easy peasy ...
{% highlight clojure %}
user=> (contains? #{:foo :bar} :foo)
true
{% endhighlight %}

How about a map?
{% highlight clojure %}
user=> (contains? {:foo 1 :bar 2} :foo)
true
{% endhighlight %}

Should work for vectors too, right?
{% highlight clojure %}
user=> (contains? [:foo :bar] :foo)
false
{% endhighlight %}

Nope. That does not work because `contains?` checks for the presence of keys, not values. And when you think about it, the keys for a vector are its indices. This is because a vector is strictly ordered and the key `1` should correspond to the second element, if any.
{% highlight clojure %}
(contains? [:foo :bar] 1)
true
{% endhighlight %}

This allows the `contains?` function to perform in constant or logarithmic time by searching only within the key indices.

Clojure provides a different function to check for values in a collection.
{% highlight clojure %}
user=> (some #(= % :foo) [:foo :bar])
true
{% endhighlight %}

Or even better:
{% highlight clojure %}
user=> (some #{:foo} [:foo :bar])  ; a set acts as a predicate 
:foo ; a truthy value
{% endhighlight %}

That might give you yet another idea
{% highlight clojure %}
user=> ((set [:foo :bar]) :foo)
:foo
user=> ((set [:foo :bar]) :doo)
nil
{% endhighlight %}

