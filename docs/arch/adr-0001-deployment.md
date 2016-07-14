0001 - GeoDocker deployment
===============================

Context
-------
Currently, it is possible to start a distributed geodocker cluster manually.
It is not yet clear how to bring them up automatically at scale.
The ideal solution should be available not only for services with a rich API (like AWS or DigitalOcean)
but also for hosting providers that lack this sophisticated API and tooling.
Currently, support for providers lacking such an API is not a priority.

Decision
--------
Was decided to look at popular docker orchestration tools:
  * [ECS](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html)
  * [DCOS](http://dcos.io)
  * [Rancher](http://rancher.com)

##### ECS

ECS is a highly scalable, fast, container management service, and it is a part of AWS ecosystem.
It's incredibly easy to launch containers via ECS,
however it is not clear from the docs or from the community (on forums) how to launch distributed applications using ECS.
The problem is that it provides no internal docker network between EC2 nodes,
and it is not possible to communicate with docker containers on different nodes by some internal address.
In addition, there is no API (or such API was not found / mentioned in docs / on forums) to launch containers on different nodes.
A common use case for ECS is to launch linked containers,
to scale them (all linked containers would be just launched on separate node(s)),
and to balance requests between these nodes using load balancer.
If we want containers to communicate with each other, it is possible to forward all necessary ports to the host network manually
(a fact which spark deployment impossible) and to talk with containers as with nodes.

##### DCOS

DC/OS is a distributed operating system based on the Apache Mesos distributed systems kernel.
It has a community version, though DC/OS is mostly (it's supposed) oriented on enterprise users.
This explains the instability of its latest community version.
It has quickstart templates for  [AWS](https://mesosphere.com/amazon/) and for [DigitalOcean](https://docs.mesosphere.com/1.7/administration/installing/cloud/digitalocean/), but is not simple to install it on [some other hosting provider](https://dcos.io/docs/1.7/administration/installing/custom/) (it is not a one-liner).
For research purposes, an AWS template was used ([modified](https://gist.github.com/pomadchin/c898fb767ce4d8bb943c2794c565fa8c) to use spot instances).
DC/OS operates with mesos DNS, and it enables docker container communication on separate nodes.
[Marathon](https://mesosphere.github.io/marathon/) is used to manage docker containers.
All docker containers start with their own internal IP addresses (available via mesos DNS),
but they are not accessable by potentially exposed ports (marathon specification requires explicit ports forwarding at least to the mesos internal network).
The upshot of this is that you can't start two containers on the same mesos node with the same exposed port
and there is no in-built port forwarding (on some internal docker network).
Because of this, it is not possible to start our own dockerized Spark using Marathon.
As a fast and simple solution for deployment, it is possible to start DC/OS built-in Hadoop and Spark packages,
and to start Accumulo using [this](https://gist.github.com/pomadchin/2193ed3a10808e9368d326a0cebe393f) Marathon job specification.
To solve Marathon DNS restrictions (as a consequence of port auto forwarding),
it is possible to use [Calico](https://www.projectcalico.org/) (though not in the current DC/OS AWS template due to old docker version),
and [Weave](https://www.weave.works/) (still has no [Weave.Net](https://www.weave.works/products/weave-net/) package for DC/OS).
Solutions are possible but require further investigation.

##### Rancher

Rancher is a completely open source docker management system.
It has an easy [insallation](http://docs.rancher.com/rancher/latest/en/installing-rancher/installing-server/) (one-liner, per machine) and may work with AWS and DigitalOcean using their APIs.
It is possible to provide slave nodes for Rancher manually (just by running another one liner on potential slaves)
and it takes control over all docker containers on them.
Rancher includes support for multiple orchestration frameworks: Cattle (native orchestrator),
Docker Swarm, Kubernetes, Mesos (beta support).
It also provides its own DNS on top of docker bridges.
Cattle supports a sort of modified docker-compose.yml (v1) file. 
Launching a cluster using Cattle is possible via [rancher-compose](http://docs.rancher.com/rancher/v1.0/zh/rancher-compose/).
However Rancher provides DNS on top of docker bridges that causes a following problem with Spark:
Spark master listens to `localhost` / `container_name` and this name in terms of a master container is an internal _docker_ IP address (17x.xxx.xxx),
and in terms of some other container master address is an internal _rancher_ ip address (10.xxx.xxx),
which makes master not available for other containers.
A similar thing happens with Accumulo: it writes the wrong ip addresses / DNS records into Zookeeper and Accumulo master
just is not available for tablets / other Accumulo processes.
A solution is possible but requires further investigation.

Consequences
------------
A clear deployment solution is still not obvious.
ECS is an Amazon service and we can just await necessary functionality.
DC/OS Community version is _very_ unstable, and has some non-trival dns problems requiring (at the current moment) third-party libraries (Calico, Weave) and a non-trival installation proccess (generally).
Rancher looks more stable, and has a more user-friendly (simpler to understand) ui / tools.
But Rancher has it's own specific features to be explored and probably requires more time to research.
