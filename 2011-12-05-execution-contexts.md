---
layout: sip
disqus: true
title: SIP-15 - ExecutionContexts
---

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

# Migration (Or Examples?)
## Updating Other Components of scala.concurrent to Use ExecutionContexts

