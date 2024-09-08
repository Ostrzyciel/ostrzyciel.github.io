---
layout: post
title: "Making Scala go fast: avoiding allocations"
date: 2024-09-08
description: "In this post, we explore the ways in which you can avoid the overhead of allocating things on the heap in Scala 3."
tags: scala jvm performance optimization garbage-collection gotta-go-fast
giscus_comments: true
thumbnail: assets/img/blog/2024/scala_perf_part_1.png
related_publications: false
# categories: scala
---

I love Scala – it's a very elegant and powerful language. This is why I decided to use it to write a high-performance serializer for knowledge graphs ([Jelly-JVM](https://w3id.org/jelly/jelly-jvm/dev/), go check it out if you are a fan of RDF) and... oh boy, was it an adventure. I spent _way_ too much time optimizing the code to make the serializer as fast as possible, and in doing that I've learned a lot about the quirks of Scala 3 and JVM optimization, which I thought may be worth sharing with others.

**Welcome to the first post in the "gotta go fast" series about Scala 3 and JVM performance.** If you don't want to miss future posts, consider subscribing to my [RSS/Atom feed](/feed.xml).

In this post, we explore the ways in which you can avoid the overhead of allocating things on the heap, which for me turned out to be the place where I've achieved the most noticeable performance gains.

---

First things first – please remember that _premature optimization is the root of all evil_. Modern JVMs are pretty smart and can optimize a lot of things for you. <u>Do not</u> optimize your Scala code before you find a clear issue with its performance. **Many techniques described here go against "the Scala way", may make your code uglier, or make it easier to introduce bugs into your code.** But, they make it go faster. 😉 You have been warned.

With this disclaimer out of the way...

## The horror of heap allocations

At some point you may run in to a situation where your application spends the majority of CPU time allocating objects in the heap and then freeing them with the garbage collector (GC). You can check this easily with any decent profiler. Sure, the allocation mechanism in modern JVMs is pretty efficient[^1], and there is a whole science dedicated to choosing the right GC for a given job, but an allocation is an allocation, and it must come with _some_ performance cost. If you are for example allocating millions of objects per seconds, then it may be worth taking a closer look at.

OK, so we have too many allocations – the solution then is obvious: **allocate less!** But how?

## The Scala "bloat": `Option`, `Either`, `Try`

`Option[T]` was created as a functional and type-safe alternative to `null` in Java, which admittedly is one of the most annoying parts of the language. Same with `Either[T1, T2]` for unions and `Try[T]` for exceptions. Let's look at an example:

```scala
import scala.util.{Success, Try}

val cat: String = "Cat!"
val someCat: Option[String] = Some(cat)
val rightCat: Either[Unit, String] = Right(cat)
val successCat: Try[String] = Success(cat)
```

Seems pretty normal. But, this ordinary snippet contains four (!) allocations: one `String`, one `Some[String]`, one `Right[String]`, and one `Success[String]`. So, every time you simply want to say "yes, this variable really does have a value" and use `Some(value)`, you make an allocation. This can add up very, very quickly in a hot path.

You can get around this by simply not using these features of Scala. I definitely _do not_ recommend doing this for public APIs, only for internal code. Please also take care to test your code properly, because these great Scala features were designed to protect your code from bugs.

- Replace `Option[T]` with just `T`. Instead of `None` use `null`. Note that using a `None` by itself does not allocate anything new, as `None` is a Scala object (singleton), so you only save allocations on removing `Some(...)`.
- Replace `Either[T1, T2]` with a [union type](https://docs.scala-lang.org/scala3/book/types-union.html): `T1 | T2`. Scala supports these pretty well. The disadvantage is that instead of nice functional methods for accessing the right/left value of an Either, you must use `match` expressions instead. Internally, matching by type uses `isInstanceOf`, which is really fast on modern JVMs. On the other hand, union types may be seen as _nicer_ in some use cases, as they support the type algebra.
- Replace `Try[T]` with good ol' exceptions... or something else entirely, like a union type. You should not rely on exceptions for the "normal" control flow of your program, but if what you are catching really are _exceptions_, then it's probably fine.

This will make your code look more like Java than Scala – it's up to you to decide if the speedup is worth it.

## Pre-allocation

I've already mentioned that using `None` by itself is free in terms of allocations – it is allocated as a singleton, so every time you use it, you get a reference to the exact same object on the heap. You can use this trick in a few other situations as well.

### Default instance

Let's take a simple example:

```scala
case class Cat(friends: List[Cat] = List())

def getLonelyCat: Cat = Cat()
```

Each time we call the `getLonelyCat` method, a new `Cat` instance is allocated on the heap. However, as the class is immutable, this does not benefit us in any way. We could have 50 different callers obtain a new lonely cat in this way and none of them would be able to modify their instance of the `Cat`. Why not instead give them the exact same instance, saving 49 allocations? They won't realize the difference, after all...

```scala
object Cat:
  val defaultInstance: Cat = Cat()

case class Cat(friends: List[Cat] = List())

def getLonelyCat: Cat = Cat.defaultInstance
```

Here we have created a companion object to the `Cat` class, which allocates a single new lonely cat at application startup (in Java lingo: _static initializer block_). We do not allocate any new lonely cats at runtime.

### Commonly-used instances

You can take the default instance trick further by pre-allocating more than one instance of a class. This admittedly has a narrower range of use cases, but is useful where you have some frequent patterns in the objects you allocate. Let's consider a temperature sensor which reports temperature measurements in Kelvin and the status of the sensor:

```scala
enum TempSensorStatus:
  case Normal, TooCold, TooHot, Error, Unresponsive

// value is in Kelvin
case class TemperatureReading(value: Double, status: TempSensorStatus)

object TemperatureReading:
  val tooCold = TemperatureReading(0.0, TempSensorStatus.TooCold)
  val tooHot = TemperatureReading(1000.0, TempSensorStatus.TooHot)
  val error = TemperatureReading(Double.NaN, TempSensorStatus.Error)
  val unresponsive = TemperatureReading(Double.NaN, TempSensorStatus.Unresponsive)
```

When reading the temperature, you can now just return one of the pre-allocated instances, instead of allocating a new one every time.

You could also use an instance cache. When you need to allocate a new object, check if it's already in your cache. If yes, then return it to the caller. If not, allocate a new one, store it in the cache, and return to the caller. Note that this is much more complex and it may be hard to get any performance improvement out of this. A lot depends on the efficiency of your cache implementation and generally it only makes sense for large classes with many internal allocations and complex initializer logic.

## Reuse, recycle

In the previous section we exploited immutability to our advantage, but the same characteristic can also cause us headaches elsewhere. A good way to avoid allocating objects would be to just reuse what you have already allocated. But, with immutability, you can't just overwrite the old contents with new data!

### Mutable collections

Things are immutable in Scala [for a reason](https://docs.scala-lang.org/scala3/book/fp-immutable-values.html), but there are also [mutable collection types](https://docs.scala-lang.org/scala3/book/collections-classes.html) which may make sense in some use cases. So, instead of:

```scala
val twoCats: List[String] = List("Black cat", "Orange cat (smart)")
// This allocates new instance of List[String]
val threeCats: List[String] = twoCats :+ "White cat (dirty)"
```

You could use:

```scala
val cats = collection.mutable.ListBuffer("Black cat", "Orange cat (smart)")
cats += "White cat (dirty)"
```

...which optimistically saves us one allocation. Of course, if the underlying collection needs to be expanded, it will internally do some allocations.

### Mutable fields

You can also consider using mutable fields in your classes, and reusing previously allocated instances that you don't need anymore. This is useful for example in caches, where instead of allocating a new value for the cache, you can just reuse an object you already had there, as you are going to evict it anyway.

Instead of:

```scala
case class Dog(name: String)

val dogs = collection.mutable.ArrayBuffer(Dog("Rex"), Dog("Max"), Dog("Woof"))
// Replace dog at index 2 with a new one (allocation & deletion)
dogs(2) = Dog("Cat-Chaser the Dog")
```

You could do:

```scala
case class Dog(var name: String)

val dogs = Array(Dog("Rex"), Dog("Max"), Dog("Woof"))
// Replace dog at index 2 with a new one (no allocation)
dogs(2).name = "Cat-Chaser the Dog"
```

Of course, then you have to be 100% sure that you control the lifetime of these instances and that you won't, for example, rename someone's dog by accident. That would be very unethical.

## Summary

In short: the less you allocate, the better. I hope these tricks will be useful to someone, but please try to use them responsibly and only if there is a clear performance issue.

In future parts of this mini-series I will tackle topics like Scala compiler quirks, polymorphism, and inlining. Stay tuned!

## Footnotes

[^1]: I highly recommend the blog of Aleksey Shipilëv if you want to learn more about how JVM's allocation, GC, and other internals work: [https://shipilev.net/jvm/anatomy-quarks/](https://shipilev.net/jvm/anatomy-quarks/)
