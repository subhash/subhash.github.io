---
layout: post
title:  "Reader macros and why you can go wrong with them"
date:   2015-05-12
categories: datomic reader
---

While using Datomic on a [project](https://github.com/subhash/heartily), I had to transform XML data to Datomic entities. I figured the easiest way to do this will be to extract maps from the XML objects and then, add on an id pair to make them directly transactable. In other words, 

{% highlight clojure %}
  {:track-point/latitude (xml1-> z (attr :lat) #(Float/valueOf %))
  :track-point/longitude (xml1-> z (attr :lon) #(Float/valueOf %))
  :track-point/altitude (xml1-> z :extensions (keyword "gpxdata:altitude") text #(Float/valueOf %))
  :track-point/heart-rate (xml1-> z :extensions (keyword "gpxtpx:TrackPointExtension") (keyword "gpxtpx:hr") text #(Integer/valueOf %))
  :track-point/speed (xml1-> z :extensions (keyword "gpxdata:speed") text #(Float/valueOf %))
  :track-point/vertical-speed (xml1-> z :extensions (keyword "gpxdata:verticalSpeed") text #(Float/valueOf %))
  :track-point/time (xml1-> z :time text #(clojure.instant/read-instant-date %))
  :db/id #db/id [:db.part/user]}
{% endhighlight %}

I felt quite smug until I actually tried running it. That's when this happened:

{% highlight clojure %}
IllegalArgumentExceptionInfo :db.error/datoms-conflict Two datoms in the same transaction conflict
{:d1
 [17592186052196 :track-point/latitude 43.73021 13194139541089 true],
 :d2
 [17592186052196 :track-point/latitude 43.730198 13194139541089 true]}
  datomic.error/argd (error.clj:77)
{% endhighlight %}

This was a classic case of the "mental shortcut" problem - I was using `#db/id` as if it were a function generating ids just because it __behaved__ like one when used in edn files like this:

{% highlight clojure %}
 {:db/id #db/id[:db.part/db]
  :db/ident :track/name
  :db/valueType :db.type/string
  :db/cardinality :db.cardinality/one
  :db.install/_attribute :db.part/db}
 {:db/id #db/id[:db.part/db]
  :db/ident :track/track-points
  :db/valueType :db.type/ref
  :db/cardinality :db.cardinality/many
  :db.install/_attribute :db.part/db}
{% endhighlight %}

But of course, `#db/id` is not a function. It is a reader macro whose behaviour is drastically different when used in code. 





