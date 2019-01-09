---
layout: post
title: Do we really need network automation?
---

In my mind, especially after seeing how automation massively helped one of the
largest global networks (Cloudflare - my current employer), I simply cannot
conceive that a network can possibly run reliably without a form of automation.
However, there still are plenty of examples of networks running (often with
major outages) without any automation at all, yet reluctant to start adopting
automation methodologies.

I have debated the subject at many conferences and meet-ups, and I heard a
variety of weak arguments against automation, or a form of anxiety caused by
false assumptions.

In today's post I would like to share my views on some the most frequent myths
I've heard, and hopefully bust them.

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
doing the most. These are easy wins that will bring excitement in your team
seeing that automation actually works, and equally creates more time for you
to automate more. The goal is, of course, to automate everything possible, but
it's always good to see early results.

What does "everything possible" mean? We now have so many tools that provide you
enough information about what happens in your network (either developed or
extended internally, e.g., napalm-logs, Prometheus metrics, etc., or commercial
products, e.g., ThousandEyes, etc.), so the question is: what do you do with all
this data? "Watch a display and when an event occurs you execute manually a
command to apply a configuration change", is not the right answer - not only
that it conflicts with the definitions I shared above, but this process also
would rely on you to see the event at the right time and act on it before your
customers are impacted; sometimes, this might be too late. (Surely, assuming the
configuration change you deploy manually is correct and it won't impact even
more negatively your network). In my opinion, one should aim for a self-healing
system that when it detects an event also applies the necessary changes.

But there's more to it than auto-remediation: what about the boring
notifications you need to write manually (i.e., in case of BGP session flapping,
interface flapping, massive packet loss caused by your transit providers, etc.).
Additionally, the system won't always be capable to fix the issue by itself: in
this case, it can create the notifications for humans to investigate the issues
further, for example by raising a ticket.

At RIPE 77 I had a talk that might help you see what I mean:
[Three years of automating large scale networks using 
Salt](https://ripe77.ripe.net/wp-content/uploads/presentations/113-RIPE77_Three_years_of_automating_large-scale_networks_using_Salt-Mircea_Ulinic.pdf)
presents some good examples (the list can be nearly
infinite) of network automation beyond configuration management triggered by
running a command manually, i.e., automatic BGP prefix limit update when the
neighbours breach the existing limits, automatic Jira tickets raised when the
BGP password is incorrect, automatic emails sent to transit providers on high
service degradation due to packet loss, etc. You can similarly implement and
automate all of these, and many, many others for more reliable, stable, and
self-resilient networks.

This is what network automation is all about.

Automation is a just a fancy thing to be in-line with the rest of the tech
==========================================================================

Managing networks comes with a very high cost as both in terms of equipment
and human resource; if the company you're working for decided to make this
investment, it probably means that the network plays a critical role within the
organisation. With this in mind, it is probably safe to assume that the
reliability and the performances of this company highly depend on the network.
In other words, the better your infrastructure, and implicitly the network, the
better regarded is your company going to be, and the customers are certainly
going to notice that. I can give an example from the company I am currently
working for, Cloudflare: before I joined, customers, for good reasons, were
always complaining about the quality of service and frequently experiencing
service degradation. Even though this was due to external causes (in particular,
extremely poor performance of the transit providers), customers don't care about
that: they pay you to offer them good services, otherwise they'll go to your
competitors, whatever would be your reasons. In our case, the reasoning was the
low speed of reaction and the scale to manually perform configuration changes
when having to deal with external factors.
Building an automation logic that intelligently reroutes the traffic, and 
applies various other configuration changes as the business logic requires,
immediately after the external factors are detected. This is something that
humans aren't able to perform  manually, especially when the configuration
changes have to be applied in tens of places simultaneously. In fact, we've seen
the results very quickly, and the number of customers on-boarded took off, while
the amount of support tickets due to network issues just dropped.

*Disclaimer*: I am not speaking in the name of my employer; similarly, I have
not been told / paid / whatever to write these: I'm trying to use this as an
example out of my own experience: to me, it was an incredible experience and
opportunity to give a helping hand with this, and seeing the results and the
positive impact on the business, as in terms of revenue, customer satisfaction,
etc. Nevertheless, there are many other factors as well, but that's beyond the
purpose of this post.

The more reliable and flexible is your network, the more customers are going 
to trust your company. Currently that's not that case with most networks: in
fact, when something goes wrong, the famous "it's always the network" tends to
be accurate. We need to do better than this, we can do better than this, we
have all the resources to do so.

Everyone needs to learn to code
===============================

This is one of the weakest arguments against automation I've heard. No, not
everyone has to learn to code. At most, your toolset might change and might be
exposed to new a slightly new world. Eventually, instead of CLI command X, you
might execute the CLI command Y, and that's pretty much it -- but the effect of
what the command does is the key here: it goes without saying that there'll
forever be a requirement for engineers that have to deeply understand the
effects of deploying a change - be it local or global across your network - 
from a networking perspective. We need an even stronger background when you
think that the new command Y does a lot more then the previous command X (from
my previous example). Providing access to someone that doesn't fully understand
the implications, may lead to disastrous results.

We inherit most of the methodologies from the system side. If you look into the
structure of the system administration teams, you'll find out that they are
usually divided (although not always a hard split) into engineers that
continuously write code, and the others that are (fully) dedicated to
operations. There is no reason why people would assume that the existing network
operations teams would migrate over night and everyone would start writing code.
Yes, there is a high demand of people writing code for networking; but, as I
mentioned in the previous paragraph, there's an even higher demand of engineers
that understand networking.

A soft delimitation between developers and operational engineers that actively
collaborate in a continuous feedback loop, is going to win on the long term -
in my opinion. I am also saying this out of the experience I had in the last two
teams I worked with. Operational engineers don't have to write code. If they
want however, they must be encouraged, their initiative is laudable, but this
cannot possibly and it will never be enforced.

To sum this up: I don't think it's feasible to assume that writing code is ever
going be a hard requirement. I do expect however small changes in the day to day
operations, but these are completely normal. Besides, network engineers are
smart and have always been able to adapt and learn new technologies.

At the same time, I would always encourage everyone to dive into writing code -
at least for fun. It's surely a plus, at the end of the day, it's an investment
in your own skills, and widening your view, and who knows when you might
actually give be able to give a helping hand and pleasantly surprise your
colleagues, or land a better (paid) job. :-)

A vast majority of the networking tooling is written in Python. If you are
interested, I would recommend a few good resources, but not limited to:

- Kirk Byers periodically runs a nice Python course for beginners.
- Mark Lutz's [*Learning Python*](https://learning-python.com/about-lp.html):
  it is the first book I read about Python. A beautiful book, I totally enjoyed
  reading.
- Matt Harrison's courses at the [O'Reilly online learning 
  platform](https://www.safaribooksonline.com/search/?query=matt%20harrison&extended_publisher_data=true&highlight=true&is_academic_institution_account=false&source=user&include_assessments=false&include_case_studies=true&include_courses=true&include_orioles=true&include_playlists=true&formats=live%20online%20training&publishers=O%27Reilly%20Media%2C%20Inc.&sort=date_added&utm_medium=email&utm_source=topic+optin&utm_campaign=awareness&utm_content=20181229+prog+nl&mkt_tok=eyJpIjoiWW1FMk9EYzNPV1ZrTlRFeSIsInQiOiJBSDF4ZnZKcGlwU3ZtZEgwRUs0NHlLa29RY1pVcEMwT0tGMmpqdDhyN25nMWhKcHpTejA3SlBlZDhyamdJMlAxK2FUdEhXdUVyNDE2VTh3d2dLMUkwODhQS1lSWnR5V0NnQjM0N3pGd2VmNDNseGFBcEZ3Skt3ek9ScUJvRFo3biJ9)

I will similarly make time to put down some notes in this direction, sharing
some tricks to show everyone how easy it is today to build something around
the existing tooling, without requiring advanced background and a bit of will.

We'll loose our jobs
====================

No, we won't. Nobody will. In fact, all the companies that embraced automation
struggle to hire: there aren't as many candidates as open roles.

This excuse is somewhat related to the previous myth regarding the requirement
of writing code. Many people fear that automation would restrict everyone to
having to learn and write code, therefore they would be replaced. Well, I hope
that with the thoughts I shared previously, I've been able to clarify that this
scenario is surely impossible. With the risk of being pedantic, I must confess
that I have experienced that myself too: at the very beginning, I felt some
engineers slightly anxious - perhaps due to the same reason; but after some
time, seeing the potential of automation, how much it simplifies their job and
exploits more their networking skills rather than their speed-typing skills, how
easy is to deploy a configuration change on hundreds of devices instantly, and
how the network auto-detects issues before they become a real concern, they
started to love automation. If you still don't believe in this, just give it.
If you are an engineer struggling to have automation adopted by your team, look
into a different approach and offer your colleagues quick and easy wins: start
by offering solutions / tools the most painful issues you're dealing with
frequently - taking that mass out of their shoulders and putting it onto the
computer to deal with it, is surely going to be a winner. Given your business
use-case, try to make it clear that automation is not about replacing engineers.

In fact that's not event the point. I once had the chance to be listening to
Tim O'Reilly speaking at APRICOT 2017 on this exact topic. His keynote
is luckily recorded and I recommend you to watch it: it is part of the
[opening ceremony](https://youtu.be/EkS_HArfu3M), the actual keynote starting
at [1:47](https://youtu.be/EkS_HArfu3M?t=2163). A slightly different version of
the same is available at https://youtu.be/s3ha6vHapcI - a better quality of the
recording, however the APRICOT talk slightly covered some topics more specific
to the networking industry. If you don't have the time right now, don't skip it.
At the very least, bookmark it for later. The key takeaway of this speech is
that you should see the power of automation as an opportunity for more
meaningful and exciting jobs. Along the history, there are plenty of examples of
how automation transformed the world. Did we run out of jobs? On the contrary -
we still struggle to hire as many engineers as we would require (think about
this statement from the perspective of the amount of work, I wouldn't want to
diverge into a management/administrative point of view, which would be a
different argument, beyond the scope of this post).

Another interesting outcome to remember is the human and machine hybrid.
A good example is the aviation industry: it is a well known fact that pilots no
longer fly modern planes manually; instead, they are assisted by computers.
I once had this argument and I have been told: "yes, but before
introducing computers in aviation, there were 6 pilots versus only 2 or 3 now".
This is true if you limit your view to a single plane only. But let's zoom out a
bit: globally, how many pilots are there nowdays compared to 50 years ago?
Millions probably versus a few thousands (rough approximation) 50 years ago. I
think this speaks for itself, it's a matter of perspective. And - most
importantly - that could not have been the case without computers: this is the
very reason why aviation is so reliable; in result, more and more people feel
more confident to fly, and continuously increasing demand can only create more
and more jobs.

With all these mobile apps and services connected through the Internet it is no
surprise that the traffic levels are increasing much faster than ever before.
I'm not telling you a secret with this, you probably know these details better
than me; I'm taking this chance to emphasise our role in this entire machinery.
It's clear that more traffic automatically implies bigger networks, when
translates to more network devices to manage. Scaling out the human resources in
order to match the gap by continuing to operate the network manually is only
going to exponentially increase the number of human mistakes. But scaling the
teams intelligently in order to operate that network more reliably through a
form of automation is a completely different discussion. One good example of
massive continuous growth of the network size is inside the data center. Not
long ago I read Dinesh Dutt's
[BGP in the data center](http://go.cumulusnetworks.com/l/32472/2017-04-14/91d5vj):
as Dinesh pointed out, managing the data center network becomes possible only
through automation.

I think you see the parallel I am drawing here: my belief is that networks
managed by humans assisted by computers will only enable for more stable and
reliable networks, which will definitely lead to more and more job demand.

Mentioned in the previous paragraph, I would like to talk again about
automation by auto-remediation: it's a given that the machine is never actually
going to auto-remediate everything, but only a part of the problems, the rest of
them being sent to a reporting system when unable or unsure what action to take.
There is no standard where to draw the line between these, it mainly depends on
the complexity of the business logic, and a variety of other environmental
factors. But one thing is for sure: they will both co-exist, and enable us to
focus on the real issues, that the machine is unable to resolve, allowing
engineers to practice engineer work.

Another fundamentally false assumption is that jobs in the networking space
would eventually evolve in such a way that only experts in both networking and
software simultaneously would have their place. With the risk of being terribly
brutal, I find this assumption ridiculous. Out of experience, it's incredibly
hard to do both networking and software at the highest levels, at the same time
- it's close to impossible. This points somehow again to the "everyone needs to
learn to code" problem which I hope I managed to clarify already. :-)

#### We'll lose our jobs after automation is done

The truth is that there's no such thing as "automation is done". If anyone tells
you that their network is fully automated, take that with a pinch of salt. It's
extremely unlikely that anyone got that far yet - as of January 2019, I'm not
aware of anyone that has that, and never heard anyone even remotely close to
that; they may have automated configuration management fully in place - very
good, great start - but remember: automation is so much more than just
configuration management (I have already expanded on this topic above).
Automation is a continuous process that is never going to end: not only that
your network is growing, but business requirements change and expansion of the
services offered by your company are at the heart of a healthy business. To
put this in a different way, you'll never be done, you will always have to
adjust / change the automation logic you put in place sooner or later (of course,
probably not entirely, but replace old pieces of the puzzle with newer ones).
It's a never ending game. I will refer again to the system side: they call this
"DevOps"; they did this for many years already - are they done yet? No. In fact,
the number of openings is now higher than ever, specifically because there's so
much more to automate.

### The CLI is dead

I am not sure what are the origins of this myth - perhaps vendors trying to sell
new products, or just the same old features branded under a fancy label, perhaps
overly excited fanboys, but hear me out: the CLI is not dead - I am still using
it, you are still using it, we will continue using it. I have initially
understood this sentence as a metaphor interpreted as "we are not going to
depend massively on the native CLI" - which potentially, ideally, would be true.
But I was wrong: I was surprised to find out that the projected "expectations"
should be that future devices would eventually be delivered without any CLI at
all.

We inherit the automation methodologies from the server side, we are barely
following what they did decades ago. Did you hear any story about Debian,
OpenBSD, or another Unix distribution dropping their CLI because there are
automation tools allowing remote execution without requiring CLI? You probably
didn't, simply because that's not going to ever happen. :-)

I expect us - and hope - that we're going to use less and less the CLI, and
steadily migrate to the automaton tools we'll eventually have in place. But,
between this and assuming that we'll suddenly get rid of the CLI completely,
is just a fairytale with unicorns. That's even more ridiculous when one of the
vendors largely trumpeting this out, Cisco, still doesn't provide a reliable
API, particularly on some platforms such as Cisco IOS, and the CLI remains the
only option you can actually use - also for automation, sadly.

Quick results
=============

I will start with an example from the real world: when starting to build a new
house, do you expect to move in immediately after starting to build it? The
same goes with automation: think about it as a construction site - you may not
see the results and the benefits immediately, but when it's done, it's so much
better to stay inside than outside. Besides, you can actually start moving in
before it's 100% ready! ;-)

Please be patient, invest time, hire more people that have experience with
writing software - even though they may not have much experience in the
networking space, they'll learn. At the same time, your network engineers might
be interested to learn software; give them time, invest in them, sign them up
to trainings and start with the programming basics. Even though it may take a
long time, or simply they'll never write hardcore software, if they have an
interest in this direction, it's good to have a background and an understanding
of what's happening under the hood.

Waiting for the "best" tool to be built
=======================================

Are you waiting for them to build themselves? WE build the tools, and by _we_
I'm including you too.

Besides: there's no such thing as "best" tool - there are simply tools that are
good to solve a particular set of challenges, and others that perfectly resolve
a different set of challenges - and they may eventually overlap (or maybe not).
This is not a discussion about the tools, the most important is for automation
to happen!

(_Note_: I have initially worded the phrase above as "the most important is for
automation to happen, in whatever way", but I wasn't happy with this: no, not
in any form, it's important to get things right, and, in your own interest and
sanity don't reinvent the wheel. My recommendation is to use a widely adopted
framework. Personally, I have a bias towards Salt, as it's by far the most
complete and flexible I've worked with, but you should use whatever makes your
environment happy, i.e., solves *all* your requirements.)

 *None* of the existing tools would ever fit perfectly and entirely your own
environment and solve all your needs over night. I'm sorry if that's surprising
you, but that's not the case today, and it will never be: you will have to
extend their capabilities and adapt them to your own needs; eventually,
whenever possible, it would very nice to give back to the community and open
source bits of your work. This is the way that is proven to produce the quickest
results for yourself, and help driving the community efforts at the same time.
Have an extensive meeting with your team and evaluate your needs; put together a
list of requirements, then investigate which automation framework would suit your
needs best. Spend time with that, analyse carefully, and always listen to your
network. It doesn't matter that I'm always telling you how great is Salt, it
doesn't matter if your best friend is an Ansible fan: all it matters is which
one suits you the best.

Why automate
============

Besides the obvious gains in terms of speed and reliability of the configuration
changes, there's a number of other benefits including:

- Easy to audit changes, and the actual configuration the devices are running.
  If your company is interested in PCI compliance, this is a big plus.
- Peer review: a change doesn't get in without being reviewed by multiple pairs
  of eyes.
- In-line with the above, you can setup a CI/CD pipleline to automatically
  check and validate your changes.
- History: you can keep tracking of the changes, and easier follow,
  incrementally what has changed, when, and why. This is also a big win in
  tracking down the root cause of an issue introduced by a particular change.
  It is true that some platforms such as Junos offer, however it comes with some
  limitations in terms of number of steps you can look back into the history,
  the description (the reasoning) of the change, and accessing that information
  locally vs globally (i.e., you need to log into each and every device to check
  this information, while through an automated system, this information is
  centralised and immediately avaialble).
- Reuse code, and existing tooling already available.
- Yet again, it's much more than just configuration management. It's about
  making your life easier and your job more reliable, from any perspective.

Please make it happen
=====================

As 2019 just begun, I hope this post is going to help you be less afraid of
automation. If you have any other concerns, or disagree with what I said, please
leave a comment or drop me an email and I will be happy to discuss. Similarly,
if you heard other weak arguments against automation which I didn't cover, I
would love to hear them.

In the end, I would like to share a video from NANOG 71, where together with
Scott Lowe, Kirk Byers, David Barroso, Jeremy Stretch, and Jathan McCollum, we
put together a panel on network automation:

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/aQFbSovedIE" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
