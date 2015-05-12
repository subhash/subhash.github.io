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

But of course, `#db/id` is not a function. It is a reader macro whose behaviour is drastically different when used in code. To understand why we tripped on this reader macro, we'll have to start by understanding the reader.

As we all know (by now :)), the Clojure compiler works on forms and not syntax trees. It is the task of the reader to parse character streams to forms that can be compiled. This can be easily demonstrated using `read-string`

{% highlight clojure %}
user=> (type (read-string "1"))
java.lang.Long
user=> (type (read-string "\"foo\""))
java.lang.String
user=> (type (read-string "foo"))
clojure.lang.Symbol
user=> (type (read-string "(defn foo [] (println \"foo\"))"))
clojure.lang.PersistentList
{% endhighlight %}

When the reader encounters certain special characters like `' (quote)` or `; (comment)` or `^ (meta)`, the usual reader behaviour is altered. For eg, for `;`, the reader reads and ignores every character until the end of the line. The `#` character is special because the reader behaviour for handling this is determined by the following character. Apart from the usual suspects - sets `#{}` and anonymous functions `#()`, you can find other tag symbols mapped to data readers by calling `clojure.core/default-data-readers`. It returns a map of tag against the corresponding reader function that'll be invoked when the tag is encountered:

{% highlight clojure %}
user=> clojure.core/default-data-readers
{inst #'clojure.instant/read-instant-date, uuid #'clojure.uuid/default-uuid-reader}
user=> (.getTime #inst "2015-05-12T12:12:05.496-00:00")
1431432725496
user=> (.getTime (clojure.instant/read-instant-date "2015-05-12T12:12:05.496-00:00"))
1431432725496
{% endhighlight %}









