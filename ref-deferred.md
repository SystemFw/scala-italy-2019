---
title: Ref + Deferred
author: Fabio Labella (SystemFw)
theme: solarized
highlightTheme: solarized-light
revealOptions:
  transition: slide
---

# Ref + Deferred
From counters to concurrent FSMs <!-- .element: class="fragment" -->

Note:
Hi there, it's nice to be back, thank you all for sticking until the
end of conference and don't worry, I'm going to take it wasy by
talking about the two notoriously lightweight subjects of purely
functional programming and non-blocking concurrency.
Last year, I gave a talk here at Scala Italy about concurrent shared
state in pure FP, showing how we can have referentially transparent
mutation with `IO`, and how the resulting model is untangled and easy
to reason about, because you can define regions of sharing just by
passing things as arguments.
This talk is the spiritual successor to that one, I left off saying
that you can build complex concurrent abstractions for your business
logic, and today I'm going to talk about `Ref` + `Deferred`, a simple,
powerful and composable scheme for building everything, from simple
counters all the way to complex finite state machines. (fragment
reveal).
Slides: 20 so far, 15 mins
Slides: 26 before example


---

<!-- .slide: data-background="img/logo.png" -->

Note: My name is Fabio, I'm a principal software engineer at Ovo
Energy in London.

---

## About me

![](img/github.png)

Note:
I'm also an Open source developer as SystemFw, mainly as a core maintainer of fs2 and cats-effect

---

## Api Evolution

**fs2**
-  <!-- .element: class="fragment" --> 0.9 -  `Ref` via `fs2.Actor`
- <!-- .element: class="fragment" --> 0.10 - `Ref` + `Promise`
-  <!-- .element: class="fragment" --> 1.0 - `Ref` + `Deferred`

Notes:
- But let's start with a tiny bit of history, all the way back in fs2 0.9 there was this central datastructure, which was also called Ref, but was a much more complex beast backed by a custom Actor implementation,
- Then in 0.10 I redesigned our concurrency scheme, introducing basically what we have today, in the form of Ref and Promise.
- And finally before fs2 1.0, we decided to move Ref and Promise to cats-effect, and renamed Promise to Deferred.


---

## Ref[F, A]

-  <!-- .element: class="fragment" --> Purely functional mutable reference
- <!-- .element: class="fragment" --> Concurrent, lock-free
-  <!-- .element: class="fragment" --> Always contains a value
-  <!-- .element: class="fragment" --> Built on `IO` + `AtomicReference`

----

## Ref api (1)

```scala
trait Ref[A] {
  def get: IO[A]
  def set(a: A): IO[Unit]
}
object Ref {
  def of[A](a: A): IO[Ref[A]]
}
```

- <!-- .element: class="fragment" --> Created by giving an initial value
- <!-- .element: class="fragment" --> Every op wrapped in `IO` for purity
- <!-- .element: class="fragment" --> `A` is an _immutable_ structure
- <!-- .element: class="fragment" --> Real version is polymorphic in `F`

Notes:

A Ref always has a value, and as you can see, Access, setting _and_ creation are all wrapped in IO,
which ensures referential transparency when encapsulating a mutable API.
If you find this confusing, I have a whole talk just on this
Real version polymorphic in F, we'll see it in a bit

----

## Ref api(2)

```scala
def report(trace: Ref[IO, List[String]], msg: String): IO[Unit] =
  for {
    t <- trace.get
    _ <- trace.set(msg :: t)
  } yield ()
```
<!-- .element: class="fragment" -->
```scala 
def flow = for {
  trace <- Ref[IO].of(List.empty[String])
  _ <- report(trace, "one").start
  _ <- report(trace, "two").start
} yield ()
``` 
<!-- .element: class="fragment" -->

<!-- .element: class="fragment" --> Can lose updates!

Note: 
- `start` spawns a fiber
- get, get, set, set loses updates

----

## update

```scala 
trait Ref[A] {
 ...
 def update(f: A => A): IO[Unit]
}
``` 
<!-- .element: class="fragment" -->

```scala
def report(trace: Ref[IO, List[String]], msg: String): IO[Unit] =
  trace.update(t => msg :: t)
```
<!-- .element: class="fragment" -->

<!-- .element: class="fragment" --> Ref supports concurrent state

Note: 
- `update` is not the same as get and then set

----

## Ref api(3)

```scala
  def sprinter(name: String, finishLine: Ref[IO, Int]): IO[Unit] =
    for {
      _ <- finishLine.update(_ + 1)
      pos <- finishLine.get
      _ <- IO { println(s"$name arrived at position $pos")}
    } yield ()
```
<!-- .element: class="fragment" -->
```scala 
  def sprint = Ref[IO].of(0).flatMap { finishLine =>
    List(
      sprinter("A", finishLine),
      sprinter("B", finishLine),
      sprinter("C", finishLine)
    ).parSequence
  }
``` 
<!-- .element: class="fragment" -->

<!-- .element: class="fragment" --> Can get wrong order!


Note:
- parSequence starts a list of `IO` concurrenty
- A update, B update, A get will cause issue

----

## modify

```scala 
trait Ref[A] {
 ...
 def modify[B](f: A => (A, B)): IO[B]
}
``` 
<!-- .element: class="fragment" -->

```scala
def sprinter(name: String, finishLine: Ref[IO, Int]): IO[Unit] =
  for {
    pos <- finishLine.modify { old => (old + 1, old + 1) }
    _ <- IO { println(s"$name arrived at position $pos")}
  } yield ()

```
<!-- .element: class="fragment" -->

<!-- .element: class="fragment" --> `update` can be written in terms of `modify`

Note: 
- update can be written in terms of modify but not the other way around

----

## Limitations

```scala
trait Ref[A] {
 ...
 def nope(f: A => IO[A]): IO[Unit]
}
```
<!-- .element: class="fragment" -->
```scala
def update(f: A => A) = IO {
  def loop: Unit = {
    val oldState = atomicReference.get
    val newState = f(oldState)
    if (atomicReference.compareAndSet(oldState, newState)) () 
    else loop
  }
  loop
}
```
<!-- .element: class="fragment" -->

-  <!-- .element: class="fragment" --> Needs to retry `f` nondeterministically
- <!-- .element: class="fragment" --> But gives great perf and cannot deadlock

Note:
- Needs to be able to retry `f` a nondeterministic amount of times
- arbitrary IOs are not idempotent
- Much greater perf than a lock in general
- Even if a thread/fiber is not executing, the others can still make progress
- Same applies to modify

---

## Deferred[F, A]

- <!-- .element: class="fragment" --> Purely functional synchronisation
- <!-- .element: class="fragment" --> Simple one-shot semantics
- <!-- .element: class="fragment" --> Semantic blocking

----

## Semantic blocking

> The fiber is waiting, but the underlying thread is free to run many other fibers

<!-- .element: class="fragment" --> TL;DR: blocking a fiber is fine

----

## Deferred api

```scala
trait Deferred[A] {
  def get: IO[A]
  def complete(a: A): IO[Unit]
}
object Deferred {
  def apply[A]: IO[Deferred[A]]
}
```
- <!-- .element: class="fragment" --> A `Deferred` starts empty
- <!-- .element: class="fragment" --> At some point it will become full
- <!-- .element: class="fragment" --> It never becomes empty or changes again

----

## Get

- <!-- .element: class="fragment" --> on a full `Deferred`: immediately returns the value
- <!-- .element: class="fragment" --> on an empty `Deferred`: semantically blocks until a value is available
- <!-- .element: class="fragment" --> can be interrupted if needed

----

## Complete

- <!-- .element: class="fragment" --> on an empty `Deferred`: fills the `Deferred` and awakes the readers

- <!-- .element: class="fragment" --> on a full `Deferred`: fails

----

## Example: gate

```scala
class Consumer {
  def read: IO[String]
}
object Consumer {
  def setup: IO[Consumer] // slow
}

class Producer {
  def write: String => IO[Unit]
}
object Producer {
  def setup: IO[Producer] // slow
}



def consumer = for {
  c <- Consumer.setup
  msg <- c.read
  _ <- IO(println(s"Received $msg"))
} yield ()

def producer = for {
  p <- Producer.setup
  msg = "Msg A"
  _ <- p.write(msg)
  _ <- IO(println(s"Sent $msg"))
} yield ()

def prog = for {
  _ <- consumer.start
  _ <- producer.start
} yield ()
```
<!-- .element: class="fragment" -->

The message can get lost
<!-- .element: class="fragment" -->

Note:
(Fabio) Remember to scroll down
Can lose a message is the producer setup finishes before the consumer's

----

## Example: gate

```scala
def consumer(done: Deferred[IO, Unit]) = for {
  c <- Consumer.setup
  _ <- done.complete(())
  msg <- c.read
  _ <- IO(println(s"Received $msg"))
} yield ()

def producer(done: Deferred[IO, Unit]) = for {
  p <- Producer.setup
  _ <- done.get
  msg = "Msg A"
  _ <- p.write(msg)
  _ <- IO(println(s"Sent $msg"))
} yield ()

def prog = for {
  d <- Deferred[IO, Unit]
  _ <- consumer(d).start
  _ <- producer(d).start
} yield ()
```

<!-- .element: class="fragment" --> What if `Consumer.setup` fails?

----

## Example: gate

```scala
def consumer(done: Deferred[IO, Either[Throwable, Unit]]) = for {
  c <- Consumer.setup.attempt // c: Either[Throwable, Consumer]
  _ <- done.complete(c.void)
  ...
} yield ()

def producer(done: Deferred[IO, Either[Throwable, Unit]]) = for {
  p <- Producer.setup
  _ <- done.get.rethrow
  ...
} yield ()

def prog = for {
  d <- Deferred[IO, Either[Throwable, Unit]]
  _ <- consumer(d).start
  _ <- producer(d).start
} yield ()
```

Note:
Synchronisation is inherently complex
We will touch on interruption at the end

---

## Full api

```scala
trait Ref[F[_], A] {
  def get: F[A]
  def set(a: A): F[Unit]
  def modify[B](f: A => (A, B)): F[B]
}
object Ref {
  def of[F[_]: Sync, A](a: A): F[Ref[F, A]]
}
trait Deferred[F[_], A] {
  def get: F[A]
  def complete(a: A): F[Unit]
}
object Deferred {
  def apply[F[_]: Concurrent, A]: F[Deferred[F, A]]
}
```

Note:
Recap the api, fits in a page, polymorphic
As a foundation, it looks too limited (no effectful update, one shot semantics)
but there is more to it that meets the eye at first, so let's look at the philosophy
and design principles

----

## Key idea

Separate _concurrent state_ from _synchronisation_

- <!-- .element: class="fragment" --> Design the simplest primitive for both
- <!-- .element: class="fragment" --> Assemble them compositionally

----

## Key idea

- `Ref`: I want to change a value atomically
- `Deferred`: I want to wait for something to happen

----

## Key idea

- `Ref` cannot deadlock
- <!-- .element: class="fragment" --> Introduce waiting only when needed...
- <!-- .element: class="fragment" --> ... by putting `Deferred`s inside `Ref`s
- <!-- .element: class="fragment" --> `Deferred` has unidirectional waiting
- <!-- .element: class="fragment" --> Most logic happens in `S => (S, A)`

---

## Finite State Machines

- <!-- .element: class="fragment" --> There is a set of _states_
- <!-- .element: class="fragment" --> There are _concurrent inputs_
- <!-- .element: class="fragment" --> There are _transitions_ between states on inputs
- <!-- .element: class="fragment" --> _Actions_ are run after the transition is done

Note:
I have had success with a pattern inspired by finite state machines

----

## Finite State Machines

```scala
def transition: S => (S, F[C])
```
<!-- .element: class="fragment" -->
```scala
def modify[B](f: A => (A, B)): F[B]
```
<!-- .element: class="fragment" -->
```scala
modify(transition).flatten
```
<!-- .element: class="fragment" -->

----

## Aside: not Actors

- <!-- .element: class="fragment" --> Different state sharing via purity
- <!-- .element: class="fragment" --> Centralised behaviour via Streams

Note:
Unlike akka, just an aside.
We don't have to use message passing/OO, can stream our way to success
locks might be simpler in simple cases, but they scale badly with complexity.

---

## Example

```scala
trait Cached[F[_], A] {
  def get: F[A]
  def expire: F[Unit]
}
object Cached {
  def create[F[_]: Concurrent, A](fa: F[A]): F[Cached[F, A]] 
}
```
```scala
sealed trait State
case class Value(v: A) extends State
case class Updating(d: Deferred[F, Either[Throwable, A]])
   extends State
case object NoValue extends State
```
<!-- .element: class="fragment" -->

----

## Example

```scala
Ref.of[F, State](NoValue).map { state =>
 new Cached[F, A] {

  def get: F[A] =
   Deferred[F, Either[Throwable, A]].flatMap { newV =>
    state.modify {
     case st @ Value(v) => st -> v.pure[F]
     case st @ Updating(inFlight) => st -> inFlight.get.rethrow
     case NoValue => Updating(newV) -> fetch(newV).rethrow
    }.flatten
  }

  def fetch(d: Deferred[F, Either[Throwable, A]]) = for {
    r <- fa.attempt
    _ <- state.set {
       r match {
        case Left(_) => NoValue
        case Right(v) => Value(v)
      }
    }
    _ <- d.complete(r)
  }  yield r

  def expire: F[Unit] = state.update {
    case Value(_) => NoValue
    case NoValue => NoValue
    case st @ Updating(_) => st
  }
}
```

----

## Interruption

- <!-- .element: class="fragment" --> Use `bracket`, `guarantee`, `uncancelable`, `onCancel`
- <!-- .element: class="fragment" --> Restore state on interruption
- <!-- .element: class="fragment" --> Separating state and waiting helps


----

## Interruption

```scala
def get: F[A] =
 Deferred[F, Either[Throwable, A]].flatMap { newV =>
  state.modify {
   case st @ Value(v) => st -> v.pure[F]
   case st @ Updating(inFlight) => st -> inFlight.get.rethrow
   case NoValue => Updating(newV) -> fetch(newV).rethrow
  }.flatten
}

def fetch(d: Deferred[F, Either[Throwable, A]]) = for {
  r <- fa.attempt
  _ <- state.set {
     r match {
      case Left(_) => NoValue
      case Right(v) => Value(v)
    }
  }
  _ <- d.complete(r)
}  yield r

def expire: F[Unit] = state.update {
  case Value(_) => NoValue
  case NoValue => NoValue
  case st @ Updating(_) => st
}
```

----

## Interruption

```scala
def fetch(d: Deferred[F, Either[Throwable, A]]) = {
  for {
    r <- fa.attempt
    _ <- state.set {
      r match {
        case Left(_) => NoValue
        case Right(v) => Value(v)
      }
    }
    _ <- d.complete(r)
  } yield r
}.onCancel {
   state.modify {
     case st @ Value(v) => 
       st -> d.complete(v.asRight).attempt.void
     case NoValue | Updating(_) =>
       NoValue -> d.complete(error.asLeft).attempt.void
   }.flatten
}
```

Note:
interruption complexity not specific to Ref/Deferred
They help
We might change it

----

## Interruption in CE 3.0

```scala
def fetch(d: Deferred[F, Either[Throwable, A]]) = {
 def action = fa.attempt.onCancel {
   state.set(NoValue) >> d.complete(error.asLeft)
 } 
 uncancelable {
   for {
     r <- cancelable(action)
     _ <- state.set {
       r match {
         case Left(_) => NoValue
         case Right(v) => Value(v)
       }
    }
    _ <- d.complete(r)
  } yield r
 }
}
```

---

# Ref + Deferred

Your building blocks for composable concurrency
<!-- .element: class="fragment" -->


---

# Questions?

- I'm not on Twitter, reach out on Gitter @SystemFw!
- Blog and talks at [https://systemfw.org](https://systemfw.org)
- [Example code](https://github.com/SystemFw/scala-italy-2019/blob/master/Examples.scala)
