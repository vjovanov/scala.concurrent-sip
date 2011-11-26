---
layout: sip
disqus: true
title: SIP-14 - Redesign of scala.concurrent
---
Redesign of scala.concurrent into a unified substrate for different parallel frameworks


# Introduction

# ExecutionContexts
## Architecture
## Defaults and Configuration
## Termination

# Tasks, Futures, and Promises
## Architecture
## Cancellation
## Exceptions

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
