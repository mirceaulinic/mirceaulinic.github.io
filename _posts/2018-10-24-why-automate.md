---
layout: post
title: Do we really need network automation?
---

TODO: reword this
Working everyday on the same you can become biased and so did I: this is why,
when speaking to someone new, I always assume that they believe in network
automation as much as I do. But people can be very surprising sometimes. :-)

In my mind, especially after seeing how automation massively helped one of the
largest global networks (Cloudflare - my current employer), I simply cannot
conceive that a network can run reliably without a form of automation. Yet there
still are plenty of examples of network running (often with major outages)
without anything at all. And they are happy with that, and reluctant to start
adopting automation methodologies.

In today's post I would like to share my views on some of the most frequent
"arguments" / claims (you name it) against automation. With this, my only goal
is to shed some light and bust some myths.

What is automation actually?
============================

One of the laws of thought states that, in order to ensure that we're speaking
the same language is to define the terms. I have searched for several
definitions for _automation_, and here's what I found:

- _"The technique, method, or system of operating or controlling a process by highly
automatic means, as by electronic devices, reducing human intervention to a
minimum."_
- _"The technique of making an apparatus, a process, or a system operate
automatically."_, where _automatically_ means _"Having a self-acting or
self-regulating mechanism"_.

On the other hand instead, automation is often (mis)understood as _just_
configuration management. Needless to say that configuration management is
indeed a major factor, but _definitely_ not the end goal. The most
important is what's the most painful to you and the one that's most boring for
the engineers in your team.

In simpler words: consider to automate whatever is the most painful to you, or
causing the most issues to your organisation. Start automating what you hate
doing the most. These are easy wins that will equally bring excitement in your
team seeing that automation actually works, but also creates more time for you
to automate more. The goal is, of course, to automate everything possible, but
it's always good to see early results.

What does "everything possible" mean? We now have so many sources of data
that give enough information about our networks, so the question is: what do you
do with all this data? The answer is not "Watch a monitor and when an event
occurs you run a command to deploy a configuration change": not only that this
conflicts with the definitions I shared above, but this process also would rely
on you to see the event at the right time and act on it before your customers
are impacted; sometimes, this might be too late. (Surely, assuming the
configuration change you apply manually is correct and it won't impact
negatively your network). In my opinion, you should aim for a self-healing
system that when it detects an event also applies the necessary changes.

But there's so much more to it than auto-remediation: what about the boring
notifications you need to write manually (i.e., in case of BGP session flapping,
interface flapping, massive packet loss caused by your transit providers, etc.).
Similarly, not always the system will be capable to fix the issue by itself, but
it can create the notifications for humans to investigate the issues further,
for example by raising a ticket.

At RIPE77 I had a talk that might help you see what I mean: _Automatic alerting
and notifications_ presents some good examples (the list can be nearly
infinite) of network automation beyond configuration management triggered by
running a command manually, i.e., automatic BGP prefix limit update when the
neighbours breach the existing limits, automatic Jira tickets raised when the
BGP password is incorrect, automatic emails sent to transit providers on high
service degradation due to packet loss, etc. You can similarly implement and
automate all of these for a more reliable, stable, self-resilient network.
This is network automation about.

We'll loose our jobs
====================

Everyone needs to learn to code
===============================

Invest in people
================


