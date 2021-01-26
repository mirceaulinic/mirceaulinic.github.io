---
layout: post
title: Cisco IOS-XR: XML API vs. NETCONF vs. gRPC
---

In one of my previous blog posts I wrote about the
[buggy XML API on Cisco IOS-XR](https://mirceaulinic.net/2017-04-14-cisco-xr-xml-agent-fun/)
and how Cisco is not committed to fix that.

The article triggered some interesting discussions on Twitter, one of the most
interesting exchange I had with [Kristian Larsson](http://plajjan.github.io/),
under [this thread](https://twitter.com/mirceaulinic/status/852787148147130372).

Although the starting point was how Cisco acknowledges bugs and does not seem to
be motivated enough to solve them - even though the XML was not officially
declared deprecated, we have divagated from this subject.
However, Kristian's question is sound and fair to ask: Cisco recommended us to
use gRPC - available only on very new platforms, why didn't we use NETCONF
instead - available on earlier releases?

NETCONF vs. gRPC
----------------

Before going any further, let's have a brief look at several technical details.

### What is a YANG model

YANG is a data modeling language defined in [RFC6020](https://tools.ietf.org/html/rfc6020),
which describes the hierarchy of the documents sent to and received from network
devices. These documents can be represented using various languages, including JSON
or XML, and transported via channels such as NETCONF or gRPC. As an aside, David
Barroso wrote a nice and detailed article [YANG for dummies](https://napalm-automation.net/yang-for-dummies/).

A great asset of the YANG models is that they can be extended via *Deviations* -
see [RFC 6020 &para;5.6.3](https://tools.ietf.org/html/rfc6020#page-31):

> In an ideal world, all devices would be required to implement the
model exactly as defined, and deviations from the model would not be
allowed.  But in the real world, devices are often not able or
designed to implement the model as written.  For YANG-based
automation to deal with these device deviations, a mechanism must
exist for devices to inform applications of the specifics of such
deviations.

### Who provides these YANG models

There are several organisations providing YANG models, the most proeminent being
[OpenConfig](http://www.openconfig.net/), the models being available at
[https://github.com/openconfig/public](https://github.com/openconfig/public) and
[IETF](https://github.com/YangModels/yang/tree/master/standard/ietf). Both aim
to provide vendor-agnostic models, as in oposite to proprietary models written
and specific only to one particular vendor / platform.

Although the OpenConfig models are already there and YANG is flexible enough
to extend them and add platform-specifics using the *deviations* referenced
above, Cisco chose to write their own. And the divergence is outstanding:
there are different models [per platform](https://github.com/YangModels/yang/tree/master/vendor/cisco),
and [per hardware](https://github.com/YangModels/yang/tree/master/vendor/cisco/xr/622),
because we know very well that although the operating system might have the same
name, it is not actually the same software running on a different chassis. For
example, IOS-XR 6.1.2 on ASR 9000 is not the same when you run on CSR or NCS
series. I find that would make much more sense to extend the existing OpenConfig
and IETF models (whenever possible) rather than re-inventing wheels. Cisco did
that in the past with the XML agent while NETCONF was already available and it
proved a bad idea -- I would have imagined they learnt something from this.

Good news: some of latest releases have many OpenConfig models available though.

### NETCONF

NETCONF (NETwork CONFiguration protocol) is network management protocol defined
in [RFC4741](https://tools.ietf.org/html/rfc4741) and later obsolete by
[RFC6241](https://tools.ietf.org/html/rfc4741). It provides mechamisms for both
configuration management and retrieval of operational data, using RPC (Remote
Procedure Calls) in the form of request-reply XML documents.

One of the differences between RFC4741 (often referenced as NETCONF 1.0) and
RFC6241 (known as NETCONF 1.1) is that the latter standardised the usage of XML
documents modeled using YANG. The support for NETCONF 1.0 on IOS-XR has been
very poor, but it has been massively improved in the latest versions, by
leveraging NETCONF 1.1.

### gRPC

[gRPC](https://grpc.io/) is an open source RPC system developed by Google. While
NETCONF relies on SSH, therefore a persistant session till the exchange of documents
is exhausted, gRPC uses HTTP2. Another big difference is that data is serialized
using a different mechanism, protocol buffers, which is another Google product.
Without diving into too many details, for the moment, let's just point out that
gRPC is lightweight and fast.

### NETCONF and gRPC on IOS-XR

Cisco aimed to leverage the same YANG models through both transports, NETCONF
and gRPC. The data provided either as JSON when retrieved via gRPC, or
XML when using NETCONF respects the same hierarchical structures, as specified
in the corresponding YANG model.

Why not NETCONF?
----------------

As stated above, the difference bewteen gRPC and NETCONF relies
mostly on the underlying transport system used and, implicitly, their advantages
and disadvantages. From the perspective of the data exposed, there should not be
much difference.

One particular detail of NETCONF implementation is that you are not able to load
raw text config. You need to structure the XML according to the YANG models,
e.g., for NTP: it requires inspecting the
[Cisco-IOS-XR-ip-ntp-cfg](https://github.com/YangModels/yang/blob/master/vendor/cisco/xr/621/Cisco-IOS-XR-ip-ntp-cfg.yang)
YANG model and building up the XML:

```xml
<rpc>
  <load-configuration>
    <ntp xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-ip-ntp-cfg">
      <peer-vrfs>
        <peer-ipv4s>
          <address-ipv4>1.2.3.4</address-ipv4>
        </peer-ipv4s>
      </peer-vrfs>
    </ntp>
  </load-configuration>
</rpc>
```

This is very nice, elegant and the right way going forward. But it implies that
*any* configuration change must go through an XML following the YANG structure;
*if a certain field or feature is not defined in the model, you cannot configure it*.
As you can probably expect, not all the features have an YANG model associated yet.

### Blockers

IPSLA is a very important feature for us -- you can learn why by watching
[this presentation](https://www.nanog.org/sites/default/files/NANOG68%20Network%20Automation%20with%20Salt%20and%20NAPALM%20Mircea%20Ulinic%20Cloudflare%20(1).pdf),
in particular starting with slide #23.

Under the [GitHub repository](https://github.com/YangModels/yang/tree/master/vendor/cisco/xr/621)
there are few SLA models, but they are not related to IPSLA, as per Einar Nilsen's
[comment](https://github.com/YangModels/yang/issues/82#issuecomment-235872964).

Summing up this conclusion with the above, *if a model is not defined, one is
not able to retrieve the operational data for the corresponding feature*, leads
to the impossibility to configure or retrieve config & operational data of
certain features such as IPSLA. Or, to put this different, when you retrieve the
running configuration, the XML document that the device provides simply doesn't
include anything about IPSLA or any other feature lacking a YANG model, like it
wouldn't be configured at all.

There are few other blockers, but they are more specific to our environment and
they would not make much sense to be publicly exposed.

Another minor detail was rollback on demand, which I understand will be
available starting with [6.2.1](https://github.com/YangModels/yang/blob/master/vendor/cisco/xr/621/Cisco-IOS-XR-group-cfg.yang).

But wait, there's more: a proeminent downside of the proprietary models is that
you can have XML tags around raw text, like:

```xml
<rpl-route-policy>
route-policy some-policy
  if destination in (0.0.0.0/0) then
    drop
  endif
  if destination in RFC-1918 then
    drop
  endif
  if destination in (0.0.0.0/25 le 32) then
    drop
  endif
  set local-preference 171
  done
end-policy
</rpl-route-policy>
```

I forwarded these concerns to Cisco and I have been told to use the gRPC
interface instead.

Why not gRPC?
-------------

gRPC indeed tackles the problem exposed above regarding the features that don't
have an YANG model associated yet -- you can retrieve or load the data as text,
as an alternative.

gRPC, unfortunately, is not availabe to anyone, but only you are lucky enough to
be able to upgrade to very new 64 bit software. Yes, gRPC is not available on 32
bit systems, due to a "*business decision*" (whatever that is supposed to mean):

> The IOS-XR 64-bit will only be supported with RSP880/RP2 and Tomahawk line-cards.

If you don't have RSP880/RP2 and Tomahawk line-cards you can say goodbye to gRPC.

What is more frustrating is that I needed to raise a TAC case to learn about this
decision (it wasn't cleary specified in the documentation back then).

Conclusions
-----------

### TL;DR

The XML API has many features, but it's unreliable, while NETCONF,
although promising, lacks many features and turns out to be a blocker in certain
cases. gRPC, on the other hand, is excellent, but unfortunately not available
on the 32 bit flavor.

Unfortunately, for us - and many others, Cisco doesn't have any solution.

### My two cents

For the short and medium term, till the entire list of features is covered by
the YANG models:

- Ability to load text config through NETCONF.
- Counterpart: retrieve configuration as text via NETCONF.
- Similarly when retrieving operational data for a feature not having a YANG model
equivalent. Not ideal, but better than nothing.

I find very important to make gRPC available on the 32 bit software version too.

Definitely more OpenConfig and IETF YANG models rather than proprietary. Ideally,
all of them.

More flexible framework, in such a way that the user is able to load newer models
and use them without having to upgrade to a potentially more unstable version.
This might be complicated to implement, but it would be very useful.

I understand this a continous work and it requires time to get better. But it
seems that it takes a very very long time to get better.
The sad part is that many will not be able to upgrade at all, if they continue
to not support older hardware.

This writing is not indended to bring a negative publicity for Cisco or their
associates. In realistic terms, everyone at Cisco I have interacted with did their
best to help, but there's something rotten at the upper layers, whoever takes
those "business decisions". I hope this post will actually help one or another
decide what interface, if any, suits their needs, depending on the hardware and
software combination available.
