---
layout: sip
disqus: true
title: SIP-14 - Redesign of scala.concurrent
---
This SIP is part of two SIPs, which together constitute a redesign of `scala.concurrent` into a unified substrate for a variety of parallel frameworks.
This proposal focuses on futures and promises.

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
