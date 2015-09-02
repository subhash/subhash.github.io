---
layout: post
title:  "Building DSLs when you don't have macros"
date:   2015-08-28
categories: scala 
---

When I started playing with Scala, one of the worries plaguing me was the lack of macros. To be fair, there is [some effort](http://docs.scala-lang.org/overviews/macros/overview.html) in the Scala community to bring macros to the language, but it seems to me that if the language is not [homoiconic](https://en.wikipedia.org/wiki/Homoiconicity), writing macros might just mean mucking with the AST.

Anyway, while I was worrying that writing DSLs in Scala would be an ardous task, I learnt about a few nice syntactic sugar treats that promises to make my life easier. Though they are not half as powerful as macros, the design is clever and pleasing. Let me demonstrate with an example. Let's say we want to transplant a Pascal-style `repeat .. until` construct all the way to Scala (Never mind that Scala has its own `do .. while`). To be precise, we want to be able to write like this:

{% highlight scala %}
var i = 5
repeat {
  println(i)
  i -= 1
} until i <= 0
{% endhighlight %}

The first part `repeat { ... }` is easy. If you squint your eyes slightly, this looks like calling a function with a block. In Scala, objects can behave like functions if they have a method `apply` defined in them. So, we can go ahead and create a singleton object called `repeat` and define an `apply` method in it
{% highlight scala %}
 	object repeat {
 		def apply(cmd: Unit) = {
 		  cmd
 		}
 	}
{% endhighlight %}

To incorporate the `until` section, we need the `apply` method to return an object which can answer to the method `until`. Now, there are a lot of ways of doing this - starting from defining a class just for this purpose. Instead, `structural types` come to the rescue. Structural types allow for anonymous defintion of types and avoid the unnecessary boilerplate of class definition like in this case:
{% highlight scala %}
 	object repeat {
 		def apply(cmd: Unit) = new {
 			def until(cond: Boolean) = {
 				cmd
 			}
 		}
 	}
{% endhighlight %}

All that remains is to check for the condition and recurse when the condition is false
{% highlight scala %}
 	object repeat {
 		def apply(cmd: => Unit) = new {
 			def until(cond: => Boolean): Unit = {
 				cmd
 				if(cond) () else until(cond)
 			}
 		}
 	}
{% endhighlight %}

The types of `cond` and `cmd` need to be pass-by-name as they need to be evaluated each time in the loop and only when required.

With Scala's flexible syntax and numerous syntactic sugars, it is indeed possible to design a decent DSL without having to resort to macros. It is a good sign that this need has been considered while designing the language






