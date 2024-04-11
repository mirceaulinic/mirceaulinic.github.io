---
layout: post
title: APNIC course on Network Automation
bigimg: /img/apnic.png
---

From the very beginning, when I first introduced Salt to the network automation
community, one of my dreams was to create a course.

Back in 2017, together with my friend Seth House, we managed to publish a small
booklet with O'Reilly, which you can still download for free at
https://www.oreilly.com/library/view/network-automation-at/9781491992524/ (or
directly from [Cloudflare](https://www.cloudflare.com/resources/assets/slt3lc6tev37/2UixizjqVh3v7L9yhcJWMT/87c1175735d490221147e1ad5866d909/network-automation-at-scale.pdf)).

Along the way, many things happened, including COVID-19, which significantly
impacted our lives and my personal plans.

In fact, the idea of an automation course began in 2019 when  [Tashi 
Phuntsho](https://blog.apnic.net/author/tashi-phuntsho/) from APNIC proposed
collaboration. As everyone knows, APNIC has been providing high-quality
workshops and courses for a long time, so I couldn't have been more excited and
privileged about this opportunity. They have the experience and resources
(time, human, and financial) that I don't have.

Little did we know what the world was going to look like less than a year later.

We started collaborating online, debating the most important topics to cover and
how deeply we should delve. After steadily working on the materials and a
complex, scalable environment, we had a course ready by the end of 2021. 🎉

With the course ready, it was time to train the trainers who would deliver the
course to hundreds or thousands of engineers in the APAC region over the next
years to come. Being based in Europe myself and APNIC in Brisbane, Australia,
together with the travel restrictions during COVID-19 times (still in place in
2021 & 2022 in some APAC locations), as well as professional or personal
obligations on both sides, made it even more difficult to find a common time
and location to gather and finalize our goal.

APRICOT 2024
------------

Fast forward to February 2024, we managed to gather together in Bangkok for our
very first run of the network automation course:
https://2024.apricot.net/program/program#/day/1/network-automation/.

The workshop spanned over 4 full days and was packed with loads of technologies
and great content (I might be biased - but don't take my word for it, register
for the next sessions). Truthfully, we didn't manage to cover everything we
prepared. We realized that we actually needed 5 full days of training!
Needless to say, this was a lab-heavy course.
In the future, APNIC may split the course into two parts: beginner and advanced.

I firmly believe that network automation is far more than just configuration
management. In fact, configuration management was the main focus of just one of
the 17 modules of the course! I'm a big fan of event-driven automation and
orchestration [before it was cool](https://mirceaulinic.net/2017-10-19-event-driven-network-automation/).
In the course, we covered and provided hands-on experience with napalm-logs,
as well as other technologies such as Prometheus, Alertmanager, Grafana,
Elasticsearch & Kibana, NetBox, TextFSM, or my little side projects
[salt-sproxy](https://mirceaulinic.net/2019-06-17-minionless-salt-automation/)
or [ISalt](https://github.com/mirceaulinic/isalt) for debugging.
As I'm hinting here, Salt is only the starting point from which we bootstrapped
an event-driven environment with a [closed-loop](https://en.wikipedia.org/wiki/Closed-loop_controller)
that integrates with various systems to gather data, receive event
notifications, and trigger automated reactions.

All of this revolved around a complex virtualised multi-vendor environment
built on a spine-leaf topology of 10 virtual machines from vendors such as
Juniper, Arista, or Cisco. The most complex lab relied on 34 Docker containers,
with each instance allocated to a trainee.
Reproducing the same environment for over ~~20~~ ~~30~~ 40 trainees was no
simple matter, but as you may have guessed, behind the scenes, we used Salt to
manage everyone's environments. :)

Future courses
--------------

I couldn't be happier that the course is now in the skillful and experienced
hands of David Phelan, Sheryl Hermoso, Terry Sweetser, and others from the
APNIC training team. Over the next years, they will deliver this course.
In fact, the next run is just a couple of weeks away at
[SANOG 41](https://academy.apnic.net/en/events?id=a0BOc000000JOkzMAG) in Mumbai.
Make sure you register for the entire workshop duration, from 25th to 28th April 2024.

For those who utilize RSS, you may subscribe to the provided RSS feed, which
will document both past and forthcoming APNIC network automation
workshops: https://mirceaulinic.net/courses.xml.
