---
layout: post
title: "Cisco IOS-XR: the buggy XML API"
subtitle: And how Cisco does not give a damn
---

The XML API is a proprietary software written by Cisco, that is leveraged only on IOS-XR. Its goal is to facilitate the configuration or retrieval of operational data from the router.
Althought NETCONF existed long time before they have introduced the XML agent, and already deployed by few network vendors, (not surprisingly) this approach was not preferred. At the very basis, the XML agent is based on a pair of Request-Reply XML documents. For more introduction details, see [this guide](http://www.cisco.com/c/en/us/td/docs/ios_xr_sw/iosxr_r4-1/xml/programming/guide/xl41apidoc/xl41over.html). These Request and Reply have a certain XML structure, described, for example [here](www.cisco.com/c/en/us/td/docs/routers/asr9000/software/asr9k_r4-3/xml/schemas/XR_XML_Schemas_ASR9K_430.html) -- be aware, this page may crash your browser!

The story of an unsolved case
-----------------------------

On the 22nd of February 2016 I have raised a case with Cisco notifying them that you are simply unable to retrieve _any_ route learnt via BGP, using the XML agent. As of today, 14th of April 2017, thus almost 14 months later, this case is still unsolved.
Discalimer: the following have been tested on ASR9K routers running IOS-XR 5.1.3, 5.3.3, 6.0.2 and 6.1.2 32-bit having really big ammounts of routes learned through BGP, a pretty good chunk of the whole Internet table.

### What's this bug about

For the beginning let's retrive the IPv4 BGP routes:

```xml
Wed Mar 15 16:11:45.162 UTC
XML> <?xml version="1.0" encoding="UTF-8"?>
<Request MajorVersion="1" MinorVersion="0">
<Get>
    <Operational>
        <BGP>
            <Active>
                <DefaultVRF>
                    <AFTable>
                        <AF>
                            <Naming>
                                <AFName>
                                    IPv4Unicast
                                </AFName>
                            </Naming>
                            <PathTable>
                            </PathTable>
                        </AF>
                    </AFTable>
                </DefaultVRF>
            </Active>
        </BGP>
    </Operational>
</Get>
</Request>
ERROR: 0xa367a600 'XML Service Library' detected the 'fatal' condition 'The throttle on the memory usage has been reached. Please optimize the request to query smaller data.'
```

The devices retuned an error message, instead of a valid XML Reply containing the details of the routes.

I have been told that the memory was not sufficient, so I have increased to the maximum available: ```xml agent tty throttle memory 600```, without any luck. Be aware: before doing the change above, make sure the device has enough free memory avaialble; the command to check is ```show watchdog memory-state```.

The XML agent can either return the entire XML output at once, either dive it in chunks, execute a loop of ping-pong Request-Reply between the consumer and the router and build up the XML Reply on the consumer. This is called [*iteration*](http://www.cisco.com/c/en/us/td/docs/ios_xr_sw/iosxr_r4-1/xml/programming/guide/xl41apidoc/xl41iter.html#wpxref82638). Configuring the iteration size to 1024KB; the same request as previously will return a very long XML document, having many zero-ized fields, such as: prefix, prefix lenght (and this ws not a default route), neithbor address, route distinguisher etc.

Another unsuccessful approach was narrowing down the request from the entire IPv4 table to a specific prefix or routes learned from a specific neighbour (say 1.2.3.4). For the later, the XML Request looks like below:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Request MajorVersion="1" MinorVersion="0">
  <Get>
      <Operational>
          <BGP>
            <InstanceTable>
             <Instance>
                 <Naming>
                     <InstanceName>
                         default
                     </InstanceName>
                 </Naming>
                 <InstanceActive>
                  <DefaultVRF>
                    <AFTable>
                      <AF>
                        <Naming>
                          <AFName>
                            IPv4Unicast
                          </AFName>
                        </Naming>
                        <PathTable>
                          <Path>
                            <Naming>
                              <NeighborAddress>
                                1.2.3.4
                              </NeighborAddress>
                            </Naming>
                          </Path>
                        </PathTable>
                      </AF>
                    </AFTable>
                  </DefaultVRF>
                 </InstanceActive>
              </Instance>
            </InstanceTable>
          </BGP>
      </Operational>
  </Get>
</Request>
```

Software means bugs
-------------------

Bugs are an intrinsic property of every software. In other words, it is not surprising to have a bug, but the difference in mentality between companies and individuals is recongised in the way they acknowledge them and how they are able to help.

We have reported few other bugs, acknowledged by Cisco. This one seems to have been a trigger and they have landed in the higher hierarchies. Somewhere around May 2016, they simply come back to us and told that the XML agent is not supported anymore (althought there wasn't any note in the official docs stating this) and they recommend us to change our tools and start using their proprietary (again!) YANG & a handful of OpenConfig models over gRPC, supposed to be included in 6.0 - not released yet at that time. Indeed, Cisco did advise to upgrade to a software version not released yet. Although we embraced the idea of using gRPC with YANG, we were simply unable to use it because they were not even aware what versions they have released! What's even more surprising is that our contact person was not a frontline TAC engineer, but someone with a very high rank inside the company, supposed to know very well these details!

The time has passed, 6.0.1 come packed with a severe bug that caused a memory leak, 6.0.2 did not have the much trumpeted gRPC either and finally 6.1.2 has been released somewhere in November 2016 with gRPC and many other features supposed to help us. Of course, they advised to upgrade to 6.1.2 and miraculously all our problems will go away!

Oh, wait, we could not upgrade to 6.1.2, as instructed, because it does not support our linecards anymore! They literally advised us to upgrade our devices to a version that is not compatible with our chassis.

Conclusions
-----------

It is clear as daylight that this case was a terrible experience: I felt at many times like working with amateurs, not professionals representing a giant networking vendor. Their attitude is a huge disappointment, they clearly proved they were not able to provide a feasible solution for our problem. Moreover, during a live session, one of the engineers admitted they have never tested such requests on devices having a decent number of routes. In my opinion, this says a lot about their software quality.

As humans we have an obligation to respect each other; each and everyone of us has the right to be respected. I personally have an even deeper respect for engineers, no matter their employer - needles to say that Cisco's engineers are very good, but it's not about individuals here. I would love to see at least a tiny bit of this respect from this company towards us, their customers, us that pay a lot for their services. Certainly that's not the case today: it's proved by the ignorance above, or other examples of nonchalantly when they [simply drops random commands](http://blog.ipspace.net/2017/04/lets-drop-some-random-commands-shall-we.html) or [introduced breaking changes betweed minor releases](http://www.fragmentationneeded.net/2017/03/cisco-not-serious-about-network.html) and many others.
