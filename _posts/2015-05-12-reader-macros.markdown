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

Users can provide custom reader functions against tags of their choice. When a mapping of tag to function is specified in a file named `data_readers.clj` in the root of the classpath, it is incorporated within a dynamic variable called [*data-readers*](http://clojure.github.io/clojure/branch-master/clojure.core-api.html#clojure.core/*data-readers*) and used whenever the reader encounters a matching tag. Unzipping `datomic-pro-0.9.5130.jar` does indeed throw up a `data_readers.clj` whose content is:

{% highlight clojure %}
{db/id datomic.db/id-literal
 db/fn datomic.function/construct
 base64 datomic.codec/base-64-literal}
{% endhighlight %}

The function `datomic.db/id-literal` seems interesting. Let's play with it:

{% highlight clojure %}
user=> (def id (datomic.db/id-literal [:db.part/user]))
#'user/id
user=> (associative? id)
true
user=> (keys id)
(:part :idx)
user=> (vals id)
(:db.part/user -1006784)
{% endhighlight %}

So, now we know that each time the reader hits upon `#db/id`, it calls a function which returns a map-like structure to stand in as the id of the entity. The original problem should be clear now. Only when the reader __encounters__ the `#db/id` literal can it substitute it with the id object. And in our code, it encounters it only once and substitutes it with a single value which is used for all entities. Clearly not what we intended. And, therefore ..

{% highlight clojure %}
  {:track-point/latitude (xml1-> z (attr :lat) #(Float/valueOf %))
  :track-point/longitude (xml1-> z (attr :lon) #(Float/valueOf %))
  :track-point/altitude (xml1-> z :extensions (keyword "gpxdata:altitude") text #(Float/valueOf %))
  :track-point/heart-rate (xml1-> z :extensions (keyword "gpxtpx:TrackPointExtension") (keyword "gpxtpx:hr") text #(Integer/valueOf %))
  :track-point/speed (xml1-> z :extensions (keyword "gpxdata:speed") text #(Float/valueOf %))
  :track-point/vertical-speed (xml1-> z :extensions (keyword "gpxdata:verticalSpeed") text #(Float/valueOf %))
  :track-point/time (xml1-> z :time text #(clojure.instant/read-instant-date %))
  :db/id (d/tempid :db.part/user)}
{% endhighlight %}








