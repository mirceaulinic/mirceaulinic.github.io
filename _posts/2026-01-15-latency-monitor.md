---
layout: post
title: "Latency Monitor: lightweight tool for TCP and UDP monitoring"
bigimg: /img/latency-monitor.png
---

**TL;DR**

* ``latency-monitor`` measures real application-like latency over TCP and UDP
* Supports one-way delay (OWD), not just RTT
* Works over established TCP connections
* Designed for high-frequency, continuous probing
* Requires time-synchronised endpoints (NTP/PTP)

I needed a way to monitor the behaviour of connections between various endpoints over TCP and UDP. “Just run a ping”
rarely works because in real networks, it rarely measures what actually matters.

1. Most of the applications are either using TCP or UDP, whereas "ping" most often means ICMP which is neither of those. 
  So ping won't help you debug anything in reality. Or, at least, not much.
2. To have a solid understanding of what's happening you need data, a lot of data. Depending on the nature of your 
  environment, you may need very granular data points to capture short-timed events or micro-bursts. You can only capture 
  these if you're monitoring continuously.
3. Many applications are designed to establish a TCP connection, maintain that state and exchange data. If you don't
  reproduce that in your monitoring, you won't be able to appropriately evaluate the network state for its actual
  production scenarios. In other words, you need a tool that monitors your network over an established TCP connection,
  as opposed to a tool that creates a separate TCP connection for every probe.
4. The return path may not always be the same (i.e., from A to B your packets may take one route, but from B back to A, 
   it may take a different route). This is where unidirectional per-segment monitoring makes a huge difference.
5. Size matters. At least when it comes to payload. A connection between point A and B may behave differently depending 
   on the packet size. Particularly for UDP.

I'm going to refer back to these points throughout this article. With these in mind, I needed a tool that satisfied the 
following requirements:

* Monitoring over TCP & UDP.
* High precision and granularity.
* Monitor over established TCP connections.
* Provide datapoints for OWD (one-way delay), not just RTT (round-trip time).
* Customisable packet size.
* Continuous monitoring and data storage using modern technologies.
* Minimal external dependencies to install on any system, even on Windows (don't ask).

To clarify, OWD is the latency between point A and B, whereas RTT is the latency between A to B and back to A.
If routing, congestion, or shaping differs per direction, RTT can hide real network behaviour that directly affects
applications (sometimes completely).

I couldn’t find a tool that checked all these boxes (or at least the most important of them). So I built one.
Famous last words.

# Enter LM

And this is how ``latency-monitor`` was born: the open source project is publicly available at
https://github.com/mirceaulinic/latency-monitor/. It is written in Python, and by default it doesn't have any external 
dependencies (more on this later). Without further ado, TL;DR, ``latency-monitor`` is a small tool that can help you 
measure the latency between endpoints bidirectionally (or single directionally, as you wish), over TCP and UDP, with
high frequency probing (up to 1 millisecond gap between probes), and nanosecond precision for the measurements.

Unlike a tool like Smokeping that you install on one machine from where you monitor the world, ``latency-monitor`` needs 
to be installed on every endpoint we need to monitor. In other words, to measure the latency between A and B (or between 
B and A), then ``latency-monitor`` needs to be installed on both A and B. This comes with an important caveat: both 
A and B need to be time synchronised, ideally from the same source. If you're not running PTP, then NTP is good enough,
bonus points if you use a stratum 0 or 1 source. The more precise your time sync, the accurate results you'll have.
https://lm.mirceaulinic.net/en/latest/arch/ presents a deeper dive into this matter.

``latency-monitor`` collects the data points (i.e., the probing results), transforms them into metrics and puts them on 
an internal pipeline from the metrics can be sent to the system of your choice. Currently, the following are available 
out of the box: Datadog, Clickhouse, ZeroMQ, and Pushgateway (although the last one is a bit of a stretch for highly
frequent probing). If you use a different backend system such as InfluxDB or others, don't be shy to open a Pull 
Request to add it, and others may benefit from it. I don't have detailed docs yet, but I'll be happy to guide you. It's 
very simple though, for example the Clickhouse integration is only 50 lines of code: 
https://github.com/mirceaulinic/latency-monitor/blob/main/latency_monitor/metrics/clickhouse.py.

# LM in action

I have a bunch of virtual machines in Amsterdam, Frankfurt, London and New York. ``latency-monitor`` is installed on 
each of those, with the following configuration file ``latency.toml`` in [TOML](https://toml.io/en/) format:

```toml
name = "amsterdam"

[metrics]
backend = "clickhouse"
host = "192.168.0.1"
username = "super"
password = "secret"

[[targets]]
host = "10.0.0.2"
label = "frankfurt"

[[targets]]
host = "10.0.0.3"
label = "london"

[[targets]]
host = "10.0.0.4"
label = "new york"
```

Similar configuration on the rest, to have a full mesh between all four machines. I'll tell you to 
[RTFM](https://lm.mirceaulinic.net/en/latest/usage/) later, for now let's unpack this config:

- ``name`` provide a human friendly label for the local system. This will be used in the metrics being reported, as the 
  ``source`` label (or tag).
- The ``metrics`` block defines the backend of your choice and connection details, i.e., where to send the data.
- Each ``targets`` entry provides the configuration of one probe. At minimum you should at least have the ``host`` 
  value, which can be either an IP or host name (ideally IP, otherwise it's always the DNS); and ``label`` will be 
  applied on the metrics as the ``target`` label (or tag).

Once you have that, start up the program on every monitored node:

```bash
$ latency-monitor -c /path/to/latency.toml

[2026-01-14 19:40:51,884] [INFO] Starting the metrics worker
[2026-01-14 19:40:51,886] [INFO] Starting the UDP server
[2026-01-14 19:40:51,889] [INFO] Starting the TCP server
[2026-01-14 19:40:51,893] [INFO] Starting the TCP latency process
[2026-01-14 19:40:51,897] [INFO] Starting the UDP OWD process for the targets
[2026-01-14 19:40:51,900] [INFO] Starting the TCP OWD process for the targets
```

Unless you manage to start up all nodes at the exact same time, you may see some errors. This is normal, as the program 
tries to connect to a TCP server that is not available yet. It is indeed very verbose.

If this looks too scary, you can only start UDP monitoring, maybe only for OWD monitoring:

```bash
$ latency-monitor -c /path/to/latency.toml --no-rtt --no-tcp

[2026-01-14 19:43:34,279] [INFO] Starting the metrics worker
[2026-01-14 19:43:34,282] [INFO] Starting the UDP server
[2026-01-14 19:43:34,284] [INFO] Starting the UDP OWD process for the targets
```

In either instance, the program won't return the command line, because it's designed to run continuously, not as 
a one-off command, as a daemon or system service constantly monitoring your links and reporting results.

If the ``metrics`` block is omitted from the TOML configuration file, then the results will be printed out on the 
screen:

```bash
$ latency-monitor -c /path/to/latency.toml --no-rtt --no-tcp

[2026-01-14 19:51:37,353] [INFO] Starting the metrics worker
[2026-01-14 19:51:37,355] [INFO] Starting the UDP server
[2026-01-14 19:51:37,357] [INFO] Starting the UDP OWD process for the targets
[2026-01-14 19:51:37,366] [INFO] Starting thread for UDP OWD target {'host': '10.0.0.2', 'label': 'frankfurt'}
[2026-01-14 19:51:37,365] [INFO] Starting thread for UDP OWD target {'host': '10.0.0.3', 'label': 'london'}
[2026-01-14 19:51:37,367] [INFO] Starting thread for UDP OWD target {'host': '10.0.0.4', 'label': 'new york'}
{"metric": "udp.wan.owd", "points": [[1767901898610114457, 12299437]], "tags": ["source:london", "target:amsterdam"]}
{"metric": "udp.wan.owd", "points": [[1767901899601658733, 4273239]], "tags": ["source:london", "target:amsterdam"]}
{"metric": "udp.wan.owd", "points": [[1767901900601804530, 4333480]], "tags": ["source:london", "target:amsterdam"]}
{"metric": "udp.wan.owd", "points": [[1767901901601676716, 4145393]], "tags": ["source:london", "target:amsterdam"]}
{"metric": "udp.wan.owd", "points": [[1767901902601867683, 4184896]], "tags": ["source:london", "target:amsterdam"]}
{"metric": "udp.wan.owd", "points": [[1767901903602142349, 4228736]], "tags": ["source:london", "target:amsterdam"]}
{"metric": "udp.wan.owd", "points": [[1767901904602049251, 4182410]], "tags": ["source:london", "target:amsterdam"]}
{"metric": "udp.wan.owd", "points": [[1767901905601724480, 3848630]], "tags": ["source:london", "target:amsterdam"]}
{"metric": "udp.wan.owd", "points": [[1767901910602923664, 4147812]], "tags": ["source:london", "target:amsterdam"]}
```

The metrics displayed have a list of tags, according to the labels configured for the targets and the local node, as 
well as the results and the collection timestamp in nanoseconds. While nanoseconds may seem superfluous, this can be
reduced to micro or milliseconds, depending on the capability of your backend system to store the metrics. It's always
easier to convert this way -- the other way, not really.

Then if you want to start bi-directional monitoring over UDP, just drop the ``--no-rtt`` flag:

```bash
$ latency-monitor -c /path/to/latency.toml --no-tcp

[2026-01-14 10:39:54,228] [INFO] Starting the metrics worker
[2026-01-14 10:39:54,230] [INFO] Starting the UDP server
[2026-01-14 10:39:54,232] [INFO] Starting the UDP OWD process for the targets
[2026-01-14 10:39:54,239] [INFO] Starting thread for UDP OWD target {'host': '10.0.0.2', 'label': 'frankfurt'}
[2026-01-14 10:39:54,238] [INFO] Starting thread for UDP OWD target {'host': '10.0.0.3', 'label': 'london'}
[2026-01-14 10:39:54,239] [INFO] Starting thread for UDP OWD target {'host': '10.0.0.4', 'label': 'new york'}
[2026-01-14 10:39:55,240] [ERROR] [UDP OWD client] Unable to unpack the computed UDP OWD from the server {'host': '10.0.0.3', 'label': 'london'}. Received: b''
[2026-01-14 10:39:55,241] [INFO] [UDP OWD client] Ignoring timestamp as SEQ doesn't match: expected 0, got -1
[2026-01-14 10:39:55,241] [ERROR] [UDP OWD client] Unable to unpack the computed UDP OWD from the server {'host': '10.0.0.4', 'label': 'new york'}. Received: b''
[2026-01-14 10:39:55,242] [INFO] [UDP OWD client] Ignoring timestamp as SEQ doesn't match: expected 0, got -1
[2026-01-14 10:39:55,241] [ERROR] [UDP OWD client] Unable to unpack the computed UDP OWD from the server {'host': '10.0.0.2', 'label': 'frankfurt'}. Received: b''
[2026-01-14 10:39:55,242] [INFO] [UDP OWD client] Ignoring timestamp as SEQ doesn't match: expected 0, got -1
{"metric": "udp.wan.rtt", "points": [[1767955195241949032, 0]], "tags": ["source:amsterdam", "target:london"]}
{"metric": "udp.wan.rtt", "points": [[1767955195242333548, 0]], "tags": ["source:amsterdam", "target:new york"]}
{"metric": "udp.wan.rtt", "points": [[1767955195242408038, 0]], "tags": ["source:amsterdam", "target:frankfurt"]}
```

If the other endpoints aren't started up yet, you'll see some errors like the above, because the program expects to 
receive a packet from the remote endpoint, but that was never received. In that case, the ``udp.wan.rtt`` metric will 
have the value 0, which means packet loss. You can play with the tool more, but I just wanted to highlight this 
important aspect around packet loss. By default, the tool waits 1 second for the return packet to arrive, and this can 
be, of course, customised to your needs.

Once again, a value of 0 for RTT or OWD explicitly represents packet loss, not zero latency (because such a thing
doesn't exist).

Anyway, once you have the firewall rules in place allowing you to probe over the designated TCP and/or UDP ports, and 
the ``latency-monitor`` service started up everywhere you need, you can start collecting the data.

# LM Data

The configuration above referenced [Clickhouse](https://clickhouse.com/) for the backend, as this is where I store my 
metrics. If you're not familiar with Clickhouse, you better do, it's awesome and simple to use.

Clickhouse is practically just a database, but humans need something visual to assess the network state. I'm using 
Grafana to plot the data. There's a [Clickhouse 
plugin](https://grafana.com/grafana/plugins/grafana-clickhouse-datasource/) publicly available which is pretty powerful 
and allows you to consume the metrics from Clickhouse.

Below you can see a graph representing the OWD over TCP between my mesh of four machines in Amsterdam, Frankfurt, London 
and New York:

<img src="../img/lm-tcp-owd.png" />

If you look closely, you may notice something interesting: the red line at the top represents the OWD from Frankfurt 
to New York, around 53ms. In the middle of the graph, there's a green line representing the unidirectional latency from 
New York to Frankfurt, around 29ms. While these values are within the normal ranges, it makes point (4) more evident, as 
it may highlight imbalanced routing or other issues (unless it's designed like that, i.e., it's not bug, it's a feature).

You wouldn't be able to tell that by looking at the RTT between Frankfurt and New York:

<img src="../img/lm-tcp-rtt.png" />

In either case, Frankfurt > New York > Frankfurt, as well as New York > Frankfurt > New York the latency shows similar
values, around 90ms. And you'd say that's fine. But is it really? And here's why OWD is super important, as this is how 
you can catch asymmetric routing.

You may notice a similar behaviour when looking at the UDP graphs, comparing OWD versus RTT:

<img src="../img/lm-udp-owd-rtt.png" />

It may also help you capture re-routing events, which is evident when the one-way latency changes (while RTT remains
constant), and may sometimes come together with minor packet loss:

<img src="../img/lm-owd-change.png" />

Or another example between Amsterdam and New York, although this one does record an increase on the return path as well:

<img src="../img/lm-owd-ams-nyc.png" />

Examples can be plenty, and it all depends on your use case after all. I let you ~~DDoS~~ explore the demo Grafana
dashboard I have available at http://dash.mirceaulinic.net:3000/public-dashboards/858e54dfad8141868a602b22bd8b7138.

At the bottom of the dashboard you'll see a panel titled "TCP Latency". By default, ``latency-monitor`` produces another
metric named, TCP Latency, that we haven't talked about yet. When I started researching the available open source tools,
I found this one named [``tcp-latency``](https://github.com/dgzlopes/tcp-latency). What this library does is for every 
probe initiates a separate TCP connection and immediately closes it, without waiting for the full handshake; the 
computed latency is the time it takes for this little dance. (Cynics would probably say this is a SYN flood attack tool, 
so beware how you use it). So while this doesn't provide the high precision I needed, I decided to keep this sort of 
measurements along, as it can be fruitful sometimes, e.g., if the TCP Latency is wildly different than the TCP RTT,
then you may have a problem somewhere, perhaps a firewall issue. It is expected however to be somewhat different, since 
tcp-latency creates a new connect with every probe, whereas the TCP RTT probing takes place over an established TCP 
session.

<img src="../img/lm-tcp-latency.png" />

This directly regards point (3), and the difference is clear: while the TCP RTT graph is relatively smooth, TCP 
Latency has a pretty high short term variability (jitter), although a stable long-term level. And it's normal to be like 
that. But if you're not looking at the right data, you're debugging a different problem.

# Final thoughts

While I’ve been running a variant of this code in production for over six months, `latency-monitor` is still very much
in its infancy.

Despite being small, it has already proven invaluable for understanding real network behaviour that RTT alone simply
cannot reveal. If you find it useful (or frustrating), I’d love to hear about it.

Issues, ideas, and pull requests are all welcome: https://github.com/mirceaulinic/latency-monitor/issues

The documentation still has gaps, and I’ll be working on improving it over the coming weeks and months.
Until then, happy probing.

Later.
