---
layout: post
title:  "Intercepting the Interceptors"
date:   2015-06-11
categories: pedestal 
---

[Pedestal](https://github.com/pedestal/pedestal) has a lot of [interesting](https://github.com/pedestal/pedestal/blob/master/guides/documentation/service-interceptors.md) ideas on how to improve over ring middleware when it comes to processing web requests. But recently, I came across a [blog post](https://stuarth.github.io/clojure/pedestal-browser-repl/) that exploited the idea even further. Before I delve more into the subject, let me briefly introduce the concepts involved. 

In Pedestal, *handlers* are functions that accept a request in the form of a map and produce a response map. These are identical with the ring handlers. But instead of ring middlewares, which are composed functions that wrap handlers, we have *interceptors* in Pedestal. Interceptors work on a given context map and can control the processing (and even termination) of the request by simply returning a modified context. Interceptors are placed in a queue and invoked one after another and the response is extracted from the final context. (Actually, Pedestal treats handlers the same way as interceptors, but we can gloss over that detail for now).

So, in Stuart Hinson's [blog post](https://stuarth.github.io/clojure/pedestal-browser-repl/), he describes an idea that allows him to respond with a browser REPL for certain requests and not others. Normally, this would be handled using an interceptor, but he does not want to tie this special behaviour based on the route hierarchy. He goes on to add this extra detail in the handler's metadata and includes an interceptor which looks into the queue for this metadata in other interceptors (and handlers). Once found, special action is taken - in this case, a browser REPL is injected into the response.

I was curious about exploring the idea with two questions:
*   Since the handler is also an `Interceptor` object, could we "add on" to the handler instead of using `meta`?
*   Could we generalize this interception so that we could wrap handlers with ad-hoc functionality during route definition?

I started by writing a simple route and its corresponding handler:

{% highlight clojure %}
(def foo-handler
 (handler 
  (fn [req]
    (-> (response "<p>This is a test message</p>")
        (content-type "text/html")))))
        
(defroutes routes
  [[["/foo" {:get [:foo-route  foo-handler ]}]]])
{% endhighlight %}

Next, I needed a "wrapper" function which could enhance the handler with the special functionality, which I decided would be to pretty-print the context map on to the response.

{% highlight clojure %}
(defn debug-wrap [handler]
  (assoc handler :wrapper-fn 
    (fn [c] (update-in c [:response :body] #(str % "<pre>" (with-out-str (pprint c)) "</pre>")))))
{% endhighlight %}

This function takes a handler and adds to it, a key called `:wrapper-fn` to hold the special function that can accept a context and adapt it as necessary. Since Clojure treats records as maps, we are able to `assoc` to it directly

Now we need an interceptor which is capable of recognizing this new key in other interceptors

{% highlight clojure %}
(def wrapper-interceptor
  (interceptor 
   {:name :wrapper-interceptor
    :enter (fn [context]
             (let [wrapper (->> context :io.pedestal.impl.interceptor/queue (some :wrapper-fn))]
               (assoc context :wrapper-fn wrapper)))
    :leave (fn [context]
             (if-let [wrapper (:wrapper-fn context)]
               (wrapper context)))}))
{% endhighlight %}

Also, the route table has to change accordingly:

{% highlight clojure %}
(defroutes routes
  [[["/foo" {:get [:foo-route  (debug-wrap foo-handler) ]}
     ^:interceptors [wrapper-interceptor]]]])
{% endhighlight %}

This approach is flexible in that, we can add other "wrappers" and use them with handlers. For eg.

{% highlight clojure %}
(defn alert-wrap [handler]
  (assoc handler :wrapper-fn 
    (fn [c] (update-in c [:response :body] #(str % "<script>alert('Wrapped!');</script>")))))
{% endhighlight %}

It's interesting to see how adaptable data-centric code can be. You can find the full code [here]()
