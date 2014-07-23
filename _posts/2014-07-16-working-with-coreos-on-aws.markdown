---
layout: post
title:  "Boring technology: AWS, CoreOS, SkyDNS"
date:   2014-07-16 10:07:18
categories: tech docker coreos aws skydns service-discovery
---

I've been working with CoreOS lately, building up a test/demo environment. [The
CloudFormation (AWS) stacks from CoreOS](http://coreos.com/docs/running-coreos/cloud-providers/ec2/)
are a handy starting point. CoreOS (and fleet) are very cool in their own
right, but they are like blank canvases. Pretty quickly I wanted to customize my
cluster. Service discovery seemed a good place to start.

There's a lot of material out there about different approaches to service
discovery. My impression is that everyone is figuring this stuff out (with the
big players leading the way). Thankfully, multiple approaches that have been
proven at significant-scale have been publicly discussed. Unfortunately, they
all advocate different approaches. A few approaches of note:

- [Netflix Eureka](http://techblog.netflix.com/2012/09/eureka.html)
- [Twitter Finagle](https://blog.twitter.com/2011/finagle-a-protocol-agnostic-rpc-system)
- [Airbnb SmartStack](http://nerds.airbnb.com/smartstack-service-discovery-cloud/)
- [Spotify DNS (boring technology)](http://labs.spotify.com/2013/02/25/in-praise-of-boring-technology/)

I decided to start with DNS SRV records for service discovery, basically
following the approach described by Spotify. Why reinvent the wheel?
This led me to 2 projects that provide DNS interfaces backed by a cluster-wide,
strongly-consistent key/value datastore:
[Consul from Hashicorp](http://www.consul.io),
and [SkyDNS from Miek Gieben](http://github.com/skynetservices/skydns).

[Consul provides it's own key/value store](http://www.consul.io/intro/getting-started/kv.html).
[SkyDNS relies on etcd (as of version 2)](https://github.com/skynetservices/skydns#changes-since-version-1).
Consul advocates that effective health-checking is a core part of service
discovery, and provides richer built-in support for it. SkyDNS doesn't address
health checking at all. CoreOS relies on etcd, so the etcd cluster was already
in place, and couldn't be dropped. Consul's duplication of functionality already
present in etcd edged out its benefits. I find Consul very interesting and
intend to check it out in the future, but I decided to skip it, for now.
So I got going with SkyDNS. 

For simplicity, I went with a process-per-host approach. Each SkyDNS instance
speaks to the local etcd instance in order to answer DNS queries. This is nice
and simple because all of the services on this cluster will be running in
containers, [which can access the host via a known IP:
172.17.42.1](http://coreos.com/blog/docker-dynamic-ambassador-powered-by-etcd/#toc_4). This allows an admin to [configure the docker daemons on all CoreOS
hosts in a cluster to use this IP as the default DNS
server](https://github.com/emmanuel/coreos-skydns-cloudformation/blob/22b5a7ee480a9dae3d33727c1b20bd6e7ce30510/cloudformation-template.json#L144)
for all containers running on the cluster. In other words, all containers can
easily access the service registry via DNS: universal access, cluster-wide.

I'm now testing using [DNS-SD](http://www.dns-sd.org) as a way to structure
the DNS SRV records. So far it seems like a good approach. Combined with
default DNS search domains, containers can be easily and flexibly configured to
discover the appropriate containers (host IP & port) for all of their
dependencies. A [decent naming convention for your DNS
records](http://mnx.io/blog/a-proper-server-naming-scheme/) helps a lot here
(specifically the suggestions under "Standardized CNAME Structure").
Along the way, I realized that it's helpful to be able to easily query the host
IP from within a container, so I [configured SkyDNS to make it
easy](https://github.com/emmanuel/coreos-skydns-cloudformation/blob/22b5a7ee480a9dae3d33727c1b20bd6e7ce30510/cloudformation-template.json#L186).
From within the container, just query DNS for `local.dns.cluster.local` (or
whatever you have your SkyDNS domain set to).

I'm a big fan of not needlessly re-inventing the wheel, so this approach to
service discover appeals on that dimension. To be more specific, it is very
valuable that service discovery via DNS is universally accessible and extremely
natural/idiomatic. Happily, service registrations are just as pleasant:
[I've been using `etcdctl` calls to set the data for SkyDNS](https://github.com/emmanuel/coreos-skydns-cloudformation/blob/22b5a7ee480a9dae3d33727c1b20bd6e7ce30510/cloudformation-template.json#L186).
It is only slightly more verbose to accomplish the same thing with curl or any
other HTTP client.

It also doesn't hurt that I find it ironic, or possibly cute, that DNS-SD is
often associated with consumer technology (e.g., Apple Bonjour) rather than
datacenter-scale enterprise IT (despite the fact that it is widely used in
enterprise IT by way of Active Directory). Anyhoo, we've all got our quirks,
right?

Next up: Zookeeper & Kafka on CoreOS.

EDIT: [Zookeeper and Kafka notes are now
published]({% post_url 2014-07-21-configuring-zookeeper-on-coreos--zk-and-kafka-part-1 %}).
