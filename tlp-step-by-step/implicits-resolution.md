---
title: Implicits Resolution
author: Luigi
date: 2015-11-07 
updated: 2016-01-22
---

Implicits are a very important feature in Scala, 
they are used basically in 2 main ways:

  - implicit parameters 
  - implicit conversions 

The implicit conversions are very useful in some cases, however 
not too interesting for what I'm going to talk about
so I will focus only on implicit parameters here, 
this is an interesting article about them [http://www.cakesolutions.net/teamblogs/demystifying-implicits-and-typeclasses-in-scala](http://www.cakesolutions.net/teamblogs/demystifying-implicits-and-typeclasses-in-scala)

## Implicit Parameters 

This is what is interesting for us because implicit parameters are heavily used to do TLP in Scala, so we need to understand how they work.

The basic idea of implicit parameters is simple, 
we declare a function that takes an implicit parameter and 
if there is a unique valid implicit instance available in the scope,
the function will take it automatically and we won't need to pass it explicitly.

```
implicit val value = 3
def foo(implicit i: Int) = println(i)
foo
```

This will print 3, nothing interesting so far, this technique is mostly used 
when we need some kind of context, a typical example is the `ExecutionContext`
that we need when we use `Future`, `Future.map` for instance  is defined 
in this way:
`def map[S](f: T => S)(implicit executor: ExecutionContext): Future[S]`

Let's go now into the interesting part, implicits parameters are resolved
also when we add type parameters:

```
trait Printer[T] {
  def print(t: T): String
}

implicit val sp: Printer[Int] = new Printer[Int] {
  def print(i :Int) = i.toString
}

def foo[T](t: T)(implicit p: Printer[T]) = p.print(t)

val res = foo(3)
// val res = foo(false)
println(s"res: ${res}")
```

From this example wee see that the compiler is able to resolve 
implicits even when there are type parameters, and indeed `foo(3)`
will work properly, while for instance `foo(true)` won't compile because 
there isn't an instance of `Printer[Boolean]` available in scope, 
note that we don't need to specify the type of `T`, thanks to the type 
inference the compiler is able to figure it out, and look for the 
appropriate `Printer` instance.

An important thing to remember is that the compiler, 
when looking for an implicit for a certain type, `Printer` in the example, 
will look automatically in his companion object, so we can rewrite this example
in this way:

```
trait Printer[T] {
  def print(t: T): String
}

object Printer {
  implicit val sp: Printer[Int] = new Printer[Int] {
    def print(i :Int) = i.toString
  }
}
```

An interesting thing is that the resolution works even if not all the type
parameters are known: 

```
trait Resolver[T, R] {
  def resolve(t: T): R
}

object Resolver {
  implicit val ib: Resolver[Int, Boolean] = new Resolver[Int, Boolean] {
    def resolve(i :Int): Boolean = i > 1
  }
  implicit val sd: Resolver[String, Double] = new Resolver[String, Double] {
    def resolve(i :String): Double = i.length.toDouble
  }
}

def foo[T, R](t: T)(implicit p: Resolver[T, R]): R = p.resolve(t)

val res1: Boolean = foo(3)
val res2: Double = foo("ciao")
```

As we can see, the function `foo` now takes two types parameters,`T` and `R`,
but only `T` is extracted from the parameter `t` that we pass, `R` is instead 
computed from the implicit resolution, and then we can use it as 
return type, `res1`  and `res2` have indeed two different types, 
depending on the input that we passed.
Using implicits here we have been able to do something similar to what
we did before with [Abstract Types](abstract-types.html), 
and we can do much more when we combine this techniques together as we will see
later.

## Type Classes 

Now that we know how to use type parameters and implicits together
we can give to this a name, this technique is called **Type Classes**, 
it is the way Polymorphism is implemented in Haskell, we can encode them in Scala as well, the most common way to do it is basically what we did before, an abstract type that takes one type parameter  
`trait Printer[T] { ... }` and different implicit implementations in the companion object,
there are however other ways to encode them, it's worth to mention: 

 - [Simulacrum](https://github.com/mpilquist/simulacrum) 
  The main goal is to avoid encoding inconsistencies using macros 
 - [Scato](https://github.com/aloiscochard/scato)
  The main goal is to avoid inheritance to encode the hierarchy of the classes using instead natural transformations 

I'm not going to explain them however here, there is a lot of great material around. 

[Demystifying Implicits and Typeclasses in Scala](http://www.cakesolutions.net/teamblogs/demystifying-implicits-and-typeclasses-in-scala)  
[The Neophyte's Guide to Scala Part 12: Type Classes](http://danielwestheide.com/blog/2013/02/06/the-neophytes-guide-to-scala-part-12-type-classes.html)

## Multi-step Resolution 

Another important aspect about implicits is that the resolution
doesn't stop at the first level, we can have implicits
that are generated taking implicit parameters themselves,
and this can go on until the compiler finds a stable implicit value,  
this is a very good way to kill the compiler!

```
trait Printer[T] {
  def print(t: T): String
}

object Printer {
  implicit val intPrinter: Printer[Int] = new Printer[Int] {
    def print(i: Int) = s"$i: Int"
  }

  implicit def optionPrinter[V](implicit pv: Printer[V]): Printer[Option[V]] =
    new Printer[Option[V]] {
      def print(ov: Option[V]) = ov match {
        case None => "None"
        case Some(v) => s"Option[${pv.print(v)}]"
      }
    }

  implicit def listPrinter[V](implicit pv: Printer[V]): Printer[List[V]] =
    new Printer[List[V]] {
      def print(ov: List[V]) = ov match {
        case Nil => "Nil"
        case l: List[V] => s"List[${l.map(pv.print).mkString(", ")}]"
      }
    }
}

def print[T](t: T)(implicit p: Printer[T]) = p.print(t)

val res = print(Option(List(1, 3, 6)))
println(s"res: ${res}")

// res: Option[List[1: Int, 3: Int, 6: Int]]
```

Let's look a this example to understand how this works,
first we defined `intPrinter` which is an implicit `val` similar to the ones 
in the previous examples, now we go to the interesting ones,
`optionPrinter` and `listPrinter` are not printer for a specific type like`Int`, 
they are printer for containers type, `Option` and `List`, which both take a type parameter. (when a type take type parameters like `List[T]` is also called type constructor)

In these cases we can define the implicit for them as `def` instead of `val` and we add a type parameter `V`, we resolve implicitly the printer for the type `V` that we use to print the content of our containers, and the compiler we'll go on resolving the implicits until it finds an implicit `val` that stops the resolution. 

Let us see what happens when we call `print(Option(List(1, 3, 6)))` in this 
example: 

 - the first type to be resolve is `Option[V]`, so we get `optionPrinter` and in this case  
   `V = List`
 - the compiler then will look for a Printer[List] and resolves `listPrinter`
   and now we have `V = Int`
    
The result is indeed `Option[List[1: Int, 3: Int, 6: Int]]`

This mechanism is very important to do type level computations, we'll see later how, remember that to do this there is compile time overhead but also runtime, because we'll have to instantiate a `Printer` instance per cycle.

