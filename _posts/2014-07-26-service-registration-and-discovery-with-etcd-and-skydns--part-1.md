---
layout: post
title: "Service registration and discovery with etcd and SkyDNS (part 1)"
date: 2014-07-26 09:04:17
categories: tech coreos etcd skydns service-registration service-discovery
---

[In the last post]({% post_url 2014-07-21-configuring-zookeeper-on-coreos--zk-and-kafka-part-1 %}),
I laid out a plan in 3 parts for conveying my notes about standing up a
Zookeeper ensemble and Kafka cluster on CoreOS. It turns out that those two are
just a piece of the puzzle, so I'm going to expand scope and (gradually) cover
all of the core services/capabilities that I implement. I'm publishing these
articles as I go, so they are not necessarily in the order that makes the most
sense. I'm learning as I go, so bear with me.

Where were we? Ah, yes... Starting from the decision to base service
registration and discovery on DNS, Zookeeper served as the pilot service for
how, specifically, this would work. As a reminder: I considered a few of the
service discovery/registration approaches [described by some leading web
companies](http://jasonwilder.com/blog/2014/02/04/service-discovery-in-the-cloud/) (in addition to discussing several of the approaches in use, in the linked
article Jason Wilder also gives a good summary of the core issues at play).
Also, I looked into some of the [emerging patterns being discussed in the
Docker community](http://www.slideshare.net/jpetazzo/shipping-applications-to-production-in-containers-with-docker/11). The 'ambassador' pattern is widely
discussed in the Docker community and may prove a winner, long-term, but it is
very shaky ground at the moment.

Additionally, several orchestration/scheduling tools have some degree of
service registration and discovery 'built-in' (e.g.,
[kubernetes](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/DESIGN.md#labels),
[Marathon/Mesos](https://github.com/mesosphere/marathon/wiki/Service-Discovery-&-Load-Balancing)). After looking around, I decided to adopt a DNS-based
approach alá that described by Spotify. As with the ambassador pattern, these
newfangled approaches may displace more traditional approaches, but as of this
writing, things are very much in flux. You could say I'm hedging my bets, but
in my mind its a question of limiting the amount of uncertainty I'm taking on
at a time.

The core of the strategy I'm adopting is the use of DNS as the service
discovery mechanism, and specifically DNS SRV records for the 'fancy' parts.
SRV records detail a pool of hosts that provide a given service within a given
domain, as well as the specific port on each host to which to make connections.
The other thing SRV records contain are priority and weight which facilitate
(simple) client-side failover and load-balancing (respectively). This looks
like:

    $ dig +short SRV _xmpp-server._tcp.gmail.com
    20 0 5269 alt4.xmpp-server.l.google.com.
    20 0 5269 alt2.xmpp-server.l.google.com.
    20 0 5269 alt1.xmpp-server.l.google.com.
    5 0 5269 xmpp-server.l.google.com.
    20 0 5269 alt3.xmpp-server.l.google.com.

The first number is the priority; lower values are higher priority. Clients are
supposed to try all higher priority servers before failing over to lower
priority ones (if any). The second number is the weight; higher values are
higher weight. Clients are supposed to randomly pick a server of the
appropriate priority, weighted according to the weight value. Third is the port
on which the service can be contacted. Finally, the last part is a canonical
hostname. As usual, [Wikipedia has the
details](http://en.wikipedia.org/wiki/SRV_record#Provisioning_for_high_service_availability).

OK, now that we're caught up, let me lay out a few design
goals/constraints/priorities I had in mind as I approached the problem:

1. Support the broadest possible range of services
2. Do not require modification (patching) of existing services
3. Services are of one of two types:
    1. cross-cutting/foundational services for the operation of the cluster itself
    2. clients of the cluster (the content)
4. Support (eventually) push-button environment provisioning, where an environment is a collection of services of type (2) as described above.
    1. Client environments should be totally isolated from each other by default
    2. Services in one client environment should be able to easily access (compose with) services in other client environments

DNS is effectively universal and partially covers point 1 by virtue of its
ubiquity. 'Partially' because support for SRV records is *not* universal, which
complicates matters for dynamic port assignments. More on this later.

In theory, we're dealing with software, so any existing software could
conceivably be adapted to any desired service discovery mechanism. In practice,
however, adapting (i.e., patching) existing services is not something to be
taken lightly. For this project, modifying services (e.g., Zookeeper) for
compatibility with the chosen service discovery mechanism is out of scope.
Basing things on DNS helps out here.

And... that's enough for this installment. I'll cover points 3 & 4 in the next
entry.
