---
layout: post
title:  "Datomic: It turtles all the way down!"
date:   2015-01-24 
categories: datomic
---


We know that the way to create an attribute is Datomic is by running a transaction with a map like this:
{% highlight clojure %}
{:db/id #db/id[:db.part/db]
 :db/ident :community/name
 :db/valueType :db.type/string
 :db/cardinality :db.cardinality/one
 :db/fulltext true
 :db/doc "A community's name"
 :db.install/_attribute :db.part/db}
{% endhighlight %}

We also know that the `:db.install/_attribute` is a back-reference to the `:db.part/db` entity which is responsible for keeping track of all the attributes in the system. Does that mean one can easily query all the attributes in the system?

{% highlight clojure %}
; datomic REPL
user=> (use '[datomic.api :only [q db] :as d])
user=> (def uri "datomic:mem://disclojures")
user=> (d/create-database uri)
user=> (def conn (d/connect uri))
user=> (q '[:find [?n ...] :where [:db.part/db :db.install/attribute ?a] [?a :db/ident ?n]] (db conn))
[:db/code :fressian/tag :db/index :db/doc :db/lang :db.excise/before :db/fn :db.install/function :db.excise/beforeT :db/excise :db/cardinality :db.install/valueType :db.install/partition :db/txInstant :db/valueType :db.excise/attrs :db/unique :db.alter/attribute :db/ident :db/noHistory :db.install/attribute :db/isComponent :db/fulltext]
{% endhighlight %}

Yes, that's right! All the system attributes (plus any others you may have defined) are modeled as values of the attribute `:db.install/attribute` of the entity `:db.part/db`. Now, that would mean that `:db.install/attribute` would be defined someplace, right? 

```clojure
user=> (q '[:find (pull ?a [*]) :where [?a :db/ident :db.install/attribute]] (db conn))
[[{:db/id 13, :db/ident :db.install/attribute, :db/valueType {:db/id 20}, :db/cardinality {:db/id 36}, :db/doc "System attribute with type :db.type/ref. Asserting this attribute on :db.part/db with value v will install v as an attribute."}]]
{% endhighlight %}

It is indeed an attribute, but the type and cardinality information is hiding behind opaque ids. No worries, entity API to the rescue!

{% highlight clojure %}
user=> (-> (db conn) (d/entity 36) (:db/ident))
:db.cardinality/many
user=> (-> (db conn) (d/entity 20) (:db/ident))
:db.type/ref
{% endhighlight %}

That makes sense. `:db.install/attribute` is a `ref` because it "points" to other attribute definitions and it's cardinality is `many` obviously because there are "many" attributes. Now, what about `:db.type/ref`?


{% highlight clojure %}
user=>  (q '[:find (pull ?a [*]) :where [?a :db/ident :db.type/ref ]] (db conn))
[[{:db/id 20, :db/ident :db.type/ref, :fressian/tag :ref, :db/doc "Value type for references. All references from one entity to another are through attributes with this value type."}]]
{% endhighlight %}

It turns out that `:db.type/ref` is one of many basic "built-in" entities from which other definitions are based on. Here are some basic attributes in Datomic:

{% highlight clojure %}
user=> (map #(-> (db conn) (d/entity %) (:db/ident)) (take 40 (range)))
(:db.part/db :db/add :db/retract :db.part/tx :db.part/user nil nil nil nil nil :db/ident :db.install/partition :db.install/valueType :db.install/attribute :db.install/function :db/excise :db.excise/attrs :db.excise/beforeT :db.excise/before :db.alter/attribute :db.type/ref :db.type/keyword :db.type/long :db.type/string :db.type/boolean :db.type/instant :db.type/fn :db.type/bytes nil nil nil nil nil nil nil :db.cardinality/one :db.cardinality/many :db.unique/value :db.unique/identity :fressian/tag)
{% endhighlight %}

It is interesting to see how Datomic's meta-system, at the very bottom, consists only of entities or facts. These basic entities are used to create a type system and instrument attribute definitions. Later, the user refers to the type entities to define attributes, which are themselves entities. 
