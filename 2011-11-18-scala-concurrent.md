---
layout: sip
disqus: true
title: SIP-14 - Redesign of scala.concurrent
---
This SIP proposes a redesign of `scala.concurrent` into a unified substrate for a variety of parallel frameworks.
Part of this proposal are futures and promises, which should be suitable as building blocks for non-blocking
concurrency libraries.

# Introduction

One of the main problems with concurrency in Scala is that, as it stands, it's impossible for different parallel
frameworks to share their thread pool. In applications that make use of multiple concurrency libraries (parallel
collections, Akka, AsyncHttpClient, ...) this leads to a proliferation of threads and reduced performance and
scalability.

# ExecutionContexts

An `ExecutionContext` provides the basic capabilities for creating and starting asynchronous `Task`s. A `Task` has a
`Future` associated with it, which is completed with the result of executing that `Task`.

## Architecture

Different parallel frameworks require different progress guarantees provided by the underlying `ExecutionContext`. For
example, tasks that potentially block the underlying thread may produce deadlocks when executed on a fixed-size thread
pool.

A concurrency library should be able to express its requirements through the type system. A generic `ExecutionContext`
may not be suitable for executing potentially blocking tasks, since it may be backed by a fixed-size thread pool.

A `BlockingExecutionContext` adds a `blocking` method for executing blocking operations. It guarantees progress if
all blocking calls are enclosed in a `blocking` block (it enables resizing the execution context to avoid deadlocks).
For example,

    blocking {
      ImageIO.read()
    }

Or,

    val x = new AnyRef
    blocking {
      x.synchronized { x.wait() }
    }

Or, 

    blocking {
      f.await() // f is a Future
    }

In the last example, we shouldn't need the `blocking` call, it should be handled by the framework.
(This possibly warrants further discussion.)

A `ForkJoinExecutionContext` is a particular type of `BlockingExecutionContext` which supports blocking when joining
a task (i.e., waiting for its termination). Its `blocking` method can be implemented using
`ManagedBlocker`s. An `UnrestrictedExecutionContext` does not impose any restrictions on the tasks it can execute.

The above mentioned types are organized in the following hierarchy:

* `BlockingExecutionContext <: ExecutionContext`
* `ForkJoinExecutionContext <: BlockingExecutionContext`
* `UnrestrictedExecutionContext <: BlockingExecutionContext`

## The ExecutionContext object

The `ExecutionContext` object provides three global execution contexts. They are configured with defaults suitable
for non-blocking, potentially blocking, and fork/join tasks, respectively.

    object ExecutionContext {
      def forNonBlocking: ExecutionContext
      def forBlocking: BlockingExecutionContext
      def forUnrestrictedBlocking: UnrestrictedExecutionContext
    }

Using one of these `ExecutionContext`s, a task can be created and started as follows:

    val task = ExecutionContext.forNonBlocking.makeTask { () => /* computation */ }
    task.start()

## Defaults and Configuration
## Termination

# Tasks, Futures, and Promises

## Architecture

`Task`s, `Future`s, and `Promise`s are separate. In concrete implementations, they may be the same classes, 
but this is not shown in the public API, and their exact types are hidden by the `makeTask` and `makePromise` 
factory methods.

The reasoning behind this is as follows; `Task`s are used internally by different frameworks, and therefore 
they have no need for monadic operations like those found in `Future`s. It must also be possible to separate 
`Task`-creation from scheduling. 

Since, the "owner" of a `Promise` has produced the value to be placed within a `Future`, thus it should not be 
necessary to call the `Future` monadic operations to access that same value. This also avoids programmer 
errors, such as calling `get` on a `Promise`, which may cause a deadlock. 

## Cancellation
## Exceptions
## Draft Proposal of the Future Trait
    trait Future[+T] {
      /** Blocks the current thread until the Future has been completed or the
       *  timeout has expired. If a timeout happens, an exception will be stored in the Future object.
       */
      def await(implicit context: ExecutionContext, timeout: Duration = ExecutionContext.timeout): Future[T]
      
      def get(implicit context: ExecutionContext, timeout: Duration = ExecutionContext.timeout): Option[T]
      
      def apply(implicit context: ExecutionContext, timeout: Duration = ExecutionContext.timeout): T
      
      def cps: ContinuationFuture[T] // We should perhaps think of a better name for this one
      
      // transformation methods
      def orElse[U >: T](other: Future[U]): Future[U]
      
      def map[U](fun: T => U): Future[U]
      
      def flatMap[U](fun: T => Future[U]): Future[U] 
      
      def flatten[U](implicit f: <:< [T, Future[U]]): Future[U]
      
      def filter(pred: T => Boolean): Future[T]
      
      def collect[U](pf: PartialFunction[T, U]): Future[U]
      
      def andThen[U](fun: T => U)(implicit evidence: MapEvidence): Future[U]
      
      def andThen[U](fun: T => Future[U])(implicit evidence: FlatMapEvidence): Future[U]
      
      def recover[U >: T](pf: PartialFunction[Throwable, U]): Future[U]
      
      /**
       * Await completion of this Future and return its value if it conforms to U's
       * erased type. Will place a ClassCastException into the Future object if the value 
       * does not conform, or a TimeoutException if it times out.
       */
      def as[U: Manifest]: Future[U] 
      
      // accessors
      def foreach[U](fun: T => U): Unit
      
      // callback methods
      def onComplete(func: Future[T] => Unit): Future.this.type
      def onFailure[U](pf: PartialFunction[Throwable, U]): Future.this.type
      def onSuccess[U](pf: PartialFunction[T, U]): Future.this.type
      
    }
## Draft Proposal of the Task Trait

    trait Task[+T] {
      def future: Future[T]
      def start(): Unit
    }

## Draft Proposal of the Promise Trait
    trait Promise[-T] {
      def fail(e: Throwable)
      def fulfill(res: T)      
    }

# Migration
## Migration From Existing Futures
## Updating Other Components of scala.concurrent to Use ExecutionContexts

# Utilities
## Scheduler
For scheduling the start of `Task`s.
## TimeOut and Duration

# Evaluation
**Goal:** Re-implement examples from the .NET Task-based Async Pattern paper \[[1][1]\], and from Finagle's documentation \[[2][2]\].

# References
1. [The Task-Based Asychronous Pattern, Stephen Toub, Microsoft, April 2011][1]
2. [Fingale Documentation][2]

  [1]: http://www.microsoft.com/download/en/details.aspx?id=19957        "NETAsync"
  [2]: http://twitter.github.com/scala_school/finagle.html  "Finagle"
