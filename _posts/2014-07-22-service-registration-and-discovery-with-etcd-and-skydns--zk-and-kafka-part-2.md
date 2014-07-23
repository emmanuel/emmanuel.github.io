---
layout: post
title:  "Configuring Zookeeper on CoreOS (ZK & Kafka part 1)"
date:   2014-07-23 09:04:17
categories: tech coreos etcd skydns service-registration service-discovery
published:   false
---

[As described in the last
post]({% post_url 2014-07-21-configuring-zookeeper-on-coreos--zk-and-kafka-part-1 %}), here is
another part of my notes about standing up a Zookeeper ensemble and Kafka
cluster on CoreOS.

The main interesting bits were the following:

1. Deciding how service announcements are going to work
2. [Configuring Zookeeper in a
    container]({% post_url 2014-07-21-configuring-zookeeper-on-coreos--zk-and-kafka-part-1 %})
3. Configure Kafka in a container

I'm taking things out of order, so this is the first part, but published
second, and I hope to publish the last one in the coming days.

Starting from the decision to base service registration and discovery on DNS, Zookeeper served as the pilot service for how, specifically, this would work. As a reminder: after considering a few of the service discovery/registration approaches [described by some leading web companies](todo-http://jason-wilder.com/blog/about/service/discovery), I decided to adopt a DNS-based approach ala that described by Spotify. The core of this is the use of DNS as the service discovery mechanism, and specifically DNS SRV records. SRV records detail a pool of hosts that provide a given service within a given domain, as well as the specific port on which to make connections. The other thing SRV records contain are priority and weight which facilitate (simple) client-side failover and load-balancing (respectively).

OK, now that we're caught up, let me lay out a few design goals I had in mind as I approached the problem:

1. DNS is universal and will allow the broadest possible range of services to participate
2. Modifying services (e.g., Zookeeper) for compatibility with a specific service discovery mechanism is out of scope
3. Eventually I intend to provide push-button environment provisioning
4. Environments should be totally isolated from each other by default
5. Services in one environment should be able to rely on (compose with) services in other environments (explicitly)
6. Services are of one of two types:
    a. cross-cutting and/or foundational services for the operation of the cluster itself
    b. clients of the cluster (the content)

