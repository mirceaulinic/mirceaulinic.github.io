---
layout: post
title: Cisco IOS-XR: XML API vs. NETCONF vs. gRPC
---

One of the most interesting discussions after publishing
[Cisco IOS-XR: the buggy XML API](https://mirceaulinic.net/2017-04-14-cisco-xr-xml-agent-fun/),
I had with Kristian Larsson, under [this thread on Twitter](https://twitter.com/mirceaulinic/status/852787148147130372).
As a side note, to see the complete details of the converstaion, you can click
on the replies of every tweet, as they are not displayed directly under the root.

Although the starting point was how Cisco acknowledges bugs and does not seem to
be motivated enough to solve them - even though the XML was not declared deprecated,
we have divagated from this subject. However, his question was very fair to ask: Cisco
recommended us to use gRPC, why didn't we use NETCONF instead?

TL;DR: because it has the same disadvantages for us.

NETCONF vs. gRPC
----------------

TODO!!!! technical details

From this perspective, the Cisco implementation does make sense:
the same models are available through both transports. The data exposed either
formatted as JSON when retrieved via gRPC, or XML when using NETCONF respects
the same hierarchical structures, as specified in the corresponding YANG model.

### What YANG models?

For IOS-XR, in particular, the YANG models available are hosted on
[GitHub](https://github.com/YangModels/yang/tree/master/vendor/cisco/xr/).
A massive drawback is that most of them are proprietary, and even more, platform
specific: IOS-XR, IOS-XE and NX-OS have different models, that may not be compatible.
Why they don't focus more on OpenConfig or IETF instead, it's still a mistery for everyone.
Given the mistake by introducing the XML API when NETCONF was already standardised
and reconsidering the decisions years after, I would imagine Cisco learnt something
and they would consider more OpenConfig and IETF rather than yet another proprietary solution!
But I can see few more OpenConfig models prepared to be included in 6.2.1 though.

Another drawback to consider was the very low developpment & release speed.

Why not NETCONF?
----------------

As stated above, the difference bewteen gRPC and NETCONF relies
mostly on the underlying transport system used and, implicitly, their advantages
and disadvantages.
From the perspective of the data exposed, there should not be much difference.

However, please note that there are some slight differences: TODO!!!!!!!!

Using the XML API, one is able to load text config using a request having the format:

```xml
<Request MajorVersion="1" MinorVersion="0">
  <CLI>
    <Exec>
      ntp peer 1.2.3.4
      object-group network ipv4 my-group 1.2.3.4/24
      interface TenGigE0/0/0/24 description dummy
    </Exec>
  </CLI>
</Request>
```

One particular detail of NETCONF is that you are not able to load raw text config,
there isn't any similar option as above. Instead, you need to structure the XML
according to the YANG models, e.g. for NTP: it requires inspecting the
[Cisco-IOS-XR-ip-ntp-cfg](https://github.com/YangModels/yang/blob/master/vendor/cisco/xr/621/Cisco-IOS-XR-ip-ntp-cfg.yang)
YANG model and building up the XML:

```xml
<rpc>
  <load-configuration>
    <ntp namespace="http://cisco.com/ns/yang/Cisco-IOS-XR-ip-ntp-cfg">
      <peer-vrfs>
        <peer-ipv4s>
          <address-ipv4>1.2.3.4</address-ipv4>
        </peer-ipv4s>
      </peer-vrfs>
    </ntp>
  </load-configuration>
</rpc>
```

This implies that any configuration change must go through an XML following
the YANG structure; *if a certain field or feature is not defined in the model,
you cannot configure it*.

Compared to the XML API, neither NETCONF or gRPC provide a way to load text config
for those features not covered yet. This is an important drawback for many, including us:
if one single detail is missing from there, it blocks your entire developpment process.

### Blockers

IPSLA is a very important feature for us -- you can learn why by watching [this presentation](https://www.nanog.org/sites/default/files/NANOG68%20Network%20Automation%20with%20Salt%20and%20NAPALM%20Mircea%20Ulinic%20Cloudflare%20(1).pdf),
in particular starting with slide #23.
This is how we monitor the backbone of the Internet and improve the web experience.

Under the [GitHub repo](https://github.com/YangModels/yang/tree/master/vendor/cisco/xr/621)
there are few SLA models, but they are not related to IPSLA, as per Einar Nilsen's
[comment](https://github.com/YangModels/yang/issues/82#issuecomment-235872964).
Again, if a model is not defined, one is not able to retrieve the operational
data for the corresponding feature.

With these said, we are not able to configure of retrieve the results of the IPSLA
probes via NETCONF/gRPC. We have thousands of probes deployed globally; without
automation it would be impossible to configure, update their details,
react to their results, and many other implications.

Only this alone would be a blocker enough to stop our efforts.

There are few others, but they are more specific to our environment and they would
not make much sense publicly, but I can say they refer to the
[configuration group model](https://github.com/YangModels/yang/blob/master/vendor/cisco/xr/621/Cisco-IOS-XR-group-cfg.yang),
which is still very poor.

Another minor detail was rollback on demand, which I understand will be
available starting with [6.2.1](https://github.com/YangModels/yang/blob/master/vendor/cisco/xr/621/Cisco-IOS-XR-group-cfg.yang).
Too bad that we won't be able to upgrade to this 64-bit version, because
they decided to not support older hardware anymore.

My two cents
------------

Briefly, the XML API has many features, but it's unreliable, while NETCONF/gRPC,
although promising, lacks many features and turns out to be a blocker in certain cases.

### Solutions

For the short and medium term, till the entire list of features is covered:

- Ability to load text config through NETCONF & gRPC.
- Counterpart: retrieve configuration as text.
- Similarly when retrieving operational data for a feature not having a YANG model
equivalent. Not ideal, but better than nothing.

Definitely more OpenConfig and IETF YANG models rather than proprietary. Ideally,
all of them (as long as they make sense for the platform) should be available.

More flexible framework, in such a way that the user is able to load newer models
and use them without having to upgrade to a potentially more unstable version.
This might be complicated to implement, but it would be very useful.

I understand that it's a continous work and it requires time to get better.
But this implies the customers to be able to upgrade with the speed they release.
How many are going to be able to upgrade so fast? Or, how many are going to
be able to upgrade at all, if they continue to not support older hardware?

Cisco has to understand the real world needs in the very first place.
Upgrading is not that easy - not as a technological challenge,
but sometimes we need to wait a certain amount of time
till the software can be considered stable. But when they acknowledge bugs and they
ignore them, when is the software going to be considered stable? Not supporting
older hardware is something desired perhaps? Do they want
to force customers to buy their newest hardware to be able to run their latest software?
If so, that would be a terrible idea, not from 2017, but more rather from the 90s.

I hope that Cisco is listening to the thoughts of the whole engineering industry
and will take better decisions, based on the customers' needs, not as they wish.
They have to listen to us, not the opposite!
