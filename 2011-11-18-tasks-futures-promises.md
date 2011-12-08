---
layout: sip
disqus: true
title: SIP-14 - Futures and Promises
---
This SIP is part of two SIPs, which together constitute a redesign of `scala.concurrent` into a unified substrate for a variety of parallel frameworks.
This proposal focuses on futures and promises.

# Outline

## Introduction h

need for futures
other frameworks developed them, we should have a unique base for all of them
Finagle futures
Akka futures briefly
Scalaz futures
Scala actors futures
Java futures and what we want to avoid (we want read-only, p-c relationship, non-blocking, asynchronous)
tasks, futures and promises and their motivational relationship
execution contexts and how do they fit in
asynchronous nature of futures - callbacks
examples here
combinators used to compose futures
examples here
blocking still possible but discouraged (blocking made difficult)
example here
Finagle example adapted


## Futures

A future is an abstraction which represents a value which may become available at some point. A future object either holds a result of a computation or an exception in the case that the computation failed for whatever reason.

The simplest way to create a future object is to invoke the `future` construct which starts an asynchronous computation and returns a future holding the result of that computation. The result becomes available once the future completes. Here is an example:

    val f = future {
      4 / 2
    }

The value 2 becomes available in the future once the computation completes. An unsuccessful computation may result in an exception. In this case, the future will hold an `ArithmeticException` instead of the value, as in the following example:

    val f = future {
      2 / 0
    }

Invoking the `future` construct uses a global execution context to start an asynchronous computation. In the case the client desires to use a custom execution context to start an asynchronous computation:

    val f = customExecutionContext future {
      4 / 2
    }

Unless the asynchronous computation was merely side-effecting, we are generally interested in the result value of the computation. To obtain a value from the future (or be thrown an exception), a client of the future would have to block until the future is completed. Although this is allowed by the `Future` API as we will show later in this document, a more preferred way to continue the computation is to register a callback on the future. This callback is called asynchronously once the future is completed. If the future has already been completed when registering the callback, then the callback may either be executed asynchronously, or sequentially on the same thread - this depends on the concrete implementation of a future.

The most general for of registering a callback is the `onComplete` method, which takes a callback function from `Either[Throwable, T]` to `U`. The `onComplete` method is parametric in the return type of the callback and it discards the result of the callback. Here is an example:

    val f = future {
      4 / 2
    }
    
    f onComplete {
      case Left(t)  => println("Failure: " + t)
      case Right(v) => println("Success: " + v)
    }
    
    println("done")

Above we use a partial function as a callback, since it provides a concise syntax.

The `onComplete` method is general in the sense that it allows the client to handle the result of both failed and successful future computations. To handle only successful computation results, the `onSuccess` callback is used:

    val f = future {
      4 / 2
    }
    
    f onSuccess {
      result => println(result)
    }

The `onSuccess` callback is only executed if the future is completed successfully. To handle only unsuccessful computation results, the `onFailure` callback is used:

    val f = future {
      2 / 0
    }
    
    f onFailure {
      case ae: ArithmeticException => println("You're dividing by 0 again.")
    } onSuccess {
      x => println("I've managed to do the impossible.")
    }

The `onFailure` callback is only executed if the future fails, that is - if it contains an exception. The above example also shows that multiple callbacks can be registered with a future.

There is another subtle difference between `onSuccess` and `onFailure`. While the `onSuccess` method takes a `Function1` callback, the `onFailure` method takes a `PartialFunction`. The benefits for this design decision are twofold.

First, since partial functions have the `isDefinedAt` method, the `onFailure` only triggers the callback if it is defined for a particular throwable. In the following example the registered callback is never triggered:

    val f = future {
      2 / 0
    }
    
    f onFailure {
      case npe: NullPointerException => println("I'd be amazed if this printed out.")
    }

Having a regular function callback as an argument to `onFailure` would require including the default case on every usage, which is cumbersome.

Second, the generalized `try-catch` block also expects a `PartialFunction` value. That means that if there are generic partial function exception handlers present in the application then they will be compatible with the `onFailure` method.

The `onTimeout` method registers callbacks triggered when the future fails with a `FutureTimeoutException`. This case can also be handled by the `onFailure` method if the partial function is defined for that exception type.

Note that all three latter on-callback methods can be expressed in terms of the `onComplete` method. As such they are by default implemented in the `scala.concurrent.Future` trait. Future implementations extending this trait must implement the `onComplete` method, but may choose to also override the other on-callback methods for performance reasons.

The examples we've shown so far tend to lend themselves naturally towards functional composition of futures.


functional composition (semantics of propagating values and exceptions)
example
projections
examples
blocking on a future
design decisions we took lead to best practices (blocking, callbacks + combinators)

## Blocking a

blockable trait
blocking contract

## Exceptions v

InterruptedException
scala.util.control.ControlThrowable
Error
ExecutionException for wrapping the three special types above
FutureTimeoutException for expressing timeouts


## Promises h

Motivating example:

val p = promise

val producer = future {
  val r = doSomething
  p fulfill r
}

val consumer = future {
  doSomethingElse
  p.future onSuccess {
    r => finish(r)
  }
}

fulfilling and breaking promises
obtaining a future from a promise


## Migration p

scala.actor.Futures?
for clients


## Implementing custom futures and promises p
for library writers


## Utils v
Timeout, Duration


# Tasks, Futures, and Promises

## Architecture

There are `Task`s, `Future`s, and `Promise`s. The main idea behind these types is that there are two ways to complete
a `Future`, by completing a `Promise`, or by executing a `Task`. `Task`s and `Promise`s are created using an
`ExecutionContext`:

    trait ExecutionContext {
      
      def makeTask[T](body: () => T)(implicit timeout: Duration): Task[T]
    
      def makePromise[T]()(implicit timeout: Duration): Promise[T]
      
    }

A `Future` can be obtained from a `Task` or from a `Promise`:

    val fut = task.future
    val fut2 = promise.future

`Task`s, `Future`s, and `Promise`s are separate. In concrete implementations, they may be the same classes, 
but this is not shown in the public API, and their exact types are hidden by the `makeTask` and `makePromise` 
factory methods.

The reasoning behind this is as follows; `Task`s are used internally by different frameworks, and therefore 
they have no need for monadic operations like those found in `Future`s. It must also be possible to separate 
`Task`-creation from scheduling. 

Since the owner of a `Promise` has produced the value to be placed within a `Future`, it should not be 
necessary to call operations of the `Future` to access that same value. This also avoids programmer 
errors, such as calling `get` on a `Promise`, which may cause a deadlock. 

## Exceptions
## The Future Trait

The following specification of the `Future` trait is based on Akka's `Future` trait. The main changes compared to
Akka are:

* All blocking methods take implicit `ExecutionContext` and `Duration` arguments.
* Methods based on continuations are moved to `ContinuationFuture`.
* There is a synonym for `flatMap` called `andThen`.

Some methods are still missing, for example, a non-blocking `isComplete` method for testing whether a future has
already been completed.

    trait Future[+T] {
      
      /**
       * Blocks the current thread until the Future has been completed or the
       * timeout has expired. If a timeout happens, an exception will be stored in the Future object.
       */
      def await(implicit context: ExecutionContext, timeout: Duration = ExecutionContext.timeout): Future[T]
      
      def get(implicit context: ExecutionContext, timeout: Duration = ExecutionContext.timeout): Option[T]
      
      def apply(implicit context: ExecutionContext, timeout: Duration = ExecutionContext.timeout): T
      
      def cps: ContinuationFuture[T] // We should perhaps think of a better name for this one
      
      // combinators
      
      def orElse[U >: T](other: Future[U]): Future[U]
      
      def map[U](fun: T => U): Future[U]
      
      def flatMap[U](fun: T => Future[U]): Future[U] 
      
      def flatten[U](implicit f: <:< [T, Future[U]]): Future[U]
      
      def filter(pred: T => Boolean): Future[T]
      
      def collect[U](pf: PartialFunction[T, U]): Future[U]
      
      def andThen[U](fun: T => U)(implicit evidence: MapEvidence): Future[U]
      
      def andThen[U](fun: T => Future[U])(implicit evidence: FlatMapEvidence): Future[U]
      
      def recover[U >: T](pf: PartialFunction[Throwable, U]): Future[U]
      
      // accessors
      
      /**
       * The contained value of this Future. Before this Future is completed
       * the value will be None. After completion the value will be Some(Right(t))
       * if it contains a valid result, or Some(Left(error)) if it contains
       * an exception.
       */
      def value: Option[Either[Throwable, T]]

      /**
       * Returns the successful result of this Future if it exists.
       */
      def result: Option[T]

      /**
       * Returns the contained exception of this Future if it exists.
       */
      def exception: Option[Throwable]

      /**
       * Await completion of this Future and return its value if it conforms to U's
       * erased type. Will place a ClassCastException into the Future object if the value 
       * does not conform, or a TimeoutException if it times out.
       */
      def as[U: Manifest](implicit context: ExecutionContext, timeout: Duration = ExecutionContext.timeout): Future[U] 
      
      def foreach[U](fun: T => U)(implicit context: ExecutionContext, timeout: Duration = ExecutionContext.timeout): Unit
      
      // callback methods

      def onComplete(func: Future[T] => Unit): Future.this.type
      def onFailure[U](pf: PartialFunction[Throwable, U]): Future.this.type
      def onSuccess[U](pf: PartialFunction[T, U]): Future.this.type
      
    }

## The Task Trait

    trait Task[+T] {
      def future: Future[T]
      def start(): Unit
    }

## The Promise Trait

    trait Promise[-T] {
      def fail(e: Throwable)
      def fulfill(res: T)      
    }

# Migration
## Migration From Existing Futures

# Utilities
## Timeouts and Duration

We propose to add Akka's `Duration` to package `scala.util`. It is not clear whether we need an additional `Timeout`
type.

# Evaluation
**Goal:** Re-implement examples from the .NET Task-based Async Pattern paper \[[1][1]\], and from Finagle's documentation \[[2][2]\].

# References
1. [The Task-Based Asychronous Pattern, Stephen Toub, Microsoft, April 2011][1]
2. [Finagle Documentation][2]

  [1]: http://www.microsoft.com/download/en/details.aspx?id=19957        "NETAsync"
  [2]: http://twitter.github.com/scala_school/finagle.html  "Finagle"
