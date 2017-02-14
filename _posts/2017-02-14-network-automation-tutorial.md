---
layout: post
title: Network automation at scale
subtitle: Up and running in 60 minutes
---

Last week I have presented at [NANOG](https://www.nanog.org/about/home) a tutorial about [Network Automation at scale: up and running in 60 minutes](http://nanog.org/meetings/abstract?id=3031). The slides are available below:

<iframe src="https://drive.google.com/file/d/0B0v5YpNF4wFSVFBFdWVTeENSS28/preview" width="100%" height="420"></iframe>

The video has been published on [YouTube](https://www.youtube.com/watch?v=99jHvkVM0Dk).

Following the steps presented in this tutorial, one should be able to start from scratch and fully automate their network(s), no matter how large, no matter how many platforms they need to support (as long as NAPALM provides the necessary driver).

I tried to emphasize:

- We are in a point where network engineers are not required anymore to learn Python or another programming language, in order to step into this world. However, this should not be misunderstood: I do support learning. My belief is that the initial steps should be easier, so everyone can begin automating. On the longer term, once the systems work for us and we have more spare time, I definitely encourage learning and contributing back to the community
- SaltStack can be used for both configuration management and orchestration
- Network automation embedded in Salt, no external modules needed
- Scalability
- Ease of setup

I have also touched the surface of several topics that I will expand more in the future, such as: states, reactors, beacons etc.
