---
layout: post
title: "Developer productivity"
category: 
tags: []
---

## Introduction

During a recent job interview I was asked "Do you think you are a 10x
developer". The concept of a "10x" developer and developer
productivity is something I have thought a lot about. Fundamentally
the hard part is figuring out what to measure. I don't have any good
answers but here is how I think about it today.

## How to measure productivity?

Since programming is a fairly creative activity it will always be
difficult to find a measure that cannot be gamed.

A simple but flawed measure is something like "lines of code" or
"features completed" or "bugs fixed". These measurements are flawed
because they are only loosely linked to the things users of the code
actually care about. In University I met someone that allegedly
completed a 5 hour coding interview in 1.5 hours with code that passed
all the unit tests. If true this is impressive and a testament to
that particular developers skills. I doubt I would ever be able to
match such a feat.

Just as a person has many personality facets, a developer can work on
different facets of productivity. I like the word facets because each
is unique while still contributing to the whole.

## Cost of programming errors

An important skill is the ability to produce "error-free" code. I
think computer programming is unique in that a single bug can cost
millions of dollars to fix. Even perfectly correct code can require
rewriting when the requirements or execution environment
changes. Examples of insanely expensive bugs include OpenSSL
HeartBleed, Intel Meltdown and more. These bugs cause the users damage
and also generate rework for the entire industry.

Programming is a continuous tradeoff between getting the code working
for a specific use-case and making it robust enough to handle multiple
use-cases. Figuring how much it will cost to develop a feature is hard
enough and the risk of expensive bug is rarely factored in. There
isn't an easy way to measure the cost of expensive bugs. The cost to
fix bugs is also hard to measure and not accounted as an engineering
cost.

Developing the skill of writing code that doesn't result in expensive
bugs often requires:

- Using tools like static analyzers, linters, enabling all compiler
  warnings, fuzzers, code quality scanners, etc. Catching errors early
  is often the best return on investment. However, each tool takes
  time to learn and integrate. Each run takes time and a high rate of
  false positives can result in lost productivity.
- Developing and maintaining a set of runtime tests. Code developed at
  the same time as tests tends to be better designed because it works
  best when dependencies are minimized. Code with a good test suite
  can be refactored more easily. On the other hand, runtime testing of
  a large software base requires significant infrastructure in order
  to minimize false positives and maintain a good feedback loop.
- Careful software reuse. Sometimes using an existing code base is the
  right thing to do. For example, almost none of the developers that
  thought they could write an encryption library have succeeded. Each
  dependency on a third party becomes a liability and has to be
  managed carefully. Ideally, it is an open source library and you can
  become part of its community and keep up with the upgrades and
  security fixes. In the worst case scenario, you end up maintaining a
  fork of the software or have to apply horrible workarounds.
- Creating operationally simple software. Even bug free software can
  be a pain to upgrade or keep operational in a high availability
  configuration. Software has many different user interfaces and one
  is how the software is installed, configured, upgraded and
  maintained. I wasn't exposed to this facet of software until I had
  to maintain a cluster of 100+ machines. I have found that whether a
  service can reload its configuration without a restart is a good
  indication if the operator interface has been taken
  seriously. Reloading configuration at runtime requires a good
  software design and test suite. When there are bugs it is too easy
  for the developers to just deprecate the feature and force
  restarts. But being able to reload a configuration without impact on
  running sessions is an operationally valuable feature.

In the wrong environment an inexperienced developer can introduce
programming errors that will cost more than their
contributions. Everyone likes to talk about "10x" programmers, but I
think we should also talk about "negative productivity" programmers
and what can be done to reduce the cost of these errors by catching
and preventing them earlier.

## Cost of fixing bugs

Debugging is a specific developer skill. It is difficult to teach and
hard to explain the instincts of a good debugger to an inexperienced
developer. Being able to make an intermittent bug easily reproducible
or use gdb to track down some memory corruption are critical skills at
the right time. I also saw a talk by a Google engineer that was
investigating a 99th percentile latency outlier and found a Linux
kernel scheduler bug that saved Google millions of dollars a year. As
systems become more complex, the bugs also become harder to fix. I
wish there were better ways to capture and train debugging expertise.

## Individual productivity versus team productivity

One of the amazing properties of software is leverage where a single
tool can make a large group of developers more productive. The goal of
every manager should also be to make their team more productive. The
goal of almost every software product is to make their customers more
productive. Being able to find and address productivity bottlenecks in
a team is another developer skill. Developing this skill often
requires:

- Understanding of the workflow of the different members of the team
- Use of automation tools to transition manual work to the computer
- Creating tools with a compelling user interface for team
- Talking with upstream and downstream teams to find ways to make
  interactions smoother and more automated

This assumes that the team works well together. A toxic team member
can reduce the productivity of an entire team. Language, timezone and
cultural differences can also hinder productivity.

## Choosing the "right" work

Even the most perfect code is useless if it doesn't solve the right
problems. Keeping development aligned with business needs can
contribute to team productivity by eliminating rework. Some of the
skills required to do this well are:

- Interacting with customers directly and understanding what their
  problems are and why they are looking to you to solve them
- Communicating technical concepts to non-technical people in an
  effective way
- Communicating non-technical requirements to technical people in an
  effective way
- Potentially developing expertise in the customer domain to
  understand their domain specific language and problem context

## Conclusion

It is impossible to be excellent in all these skills. The most
important is to constantly find ways to improve individual and team
productivity. I suspect this isn't the answer that an interviewer is
expecting. I need to come up with a shorter answer.
