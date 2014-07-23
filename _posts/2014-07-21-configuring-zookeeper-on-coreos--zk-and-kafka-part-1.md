---
layout: post
title:  "Configuring Zookeeper on CoreOS (ZK & Kafka part 1)"
date:   2014-07-21 17:34:11
categories: tech coreos zookeeper kafka
---

[As promised]({% post_url 2014-07-16-working-with-coreos-on-aws %}), here are
my notes about standing up a Zookeeper ensemble and Kafka cluster on CoreOS.

The main interesting bits were the following:

1. Deciding how service announcements are going to work
2. Configuring Zookeeper in a container
3. Configure Kafka in a container

I learned enough in each of these phases to break it out into a separate little
article. This is the second part, and I'll put together the other two in the
coming days.

Zookeeper configuration is fairly straightforward. The basic overview is that
each node in an 'ensemble' (Zookeeper terminology for a consensus
group/cluster) starts with an identical configuration that enumerates the
Zookeeper settings as well as the name (or IP) and port of all nodes in the
ensemble. Multiple identical instances? That sounds pretty straight-forward,
from a Docker container perspective. But, of course, there are some bumps along
the way.

A note about ports. I am starting from the assumption that I will only run a
single Zookeeper container per CoreOS host, and that I will not need or want to
run any other services at any of the ports Zookeeper relies on (defaults are
client: 2181, peer: 2888, and leader: 3888). In a sense, I took the approach of
'statically' allocating those ports to Zookeeper. By which I mean that I am not
even attempting to bind a random or known-open port on the CoreOS host for
Zookeeper. This (greatly) simplifies Zookeeper configuration (and helps with
Kafka, too), and doesn't seem like too much of a constraint. My (current)
thinking is to statically assign ports for all cluster-internal services.

You will need a line in the Zookeeper config file
(`/etc/zookeeper/conf/zoo.cfg` on Ubuntu) for every Zookeeper instance you plan
to run on your CoreOS cluster. Your values or count may differ, but basically,
they should look like this:

    server.1=zookeeper-1:2888:3888
    server.2=zookeeper-2:2888:3888
    server.3=zookeeper-3:2888:3888

In other words, one must pre-declare the size and membership of a Zookeeper
cluster. I find this to be an interesting contrast to using `discovery.etcd.io`
to bootstrap an etcd cluster, which is what I've been using on my CoreOS
clusters. Also, I haven't explored any Zookeeper cluster expansion or migration
scenarios, so I'm crossing my fingers that it's a solved issue.

I said earlier that the instances were configured identically. That's a bit of
a fib; there is a bit of non-identical configuration on each Zookeeper node: a
`myid` file. The content of this file is a single integer in the range 1-255,
which identifies which of the `server.X` lines apply to the current node (and
conversely, how to find peers). On Ubuntu 14.04, this file is located at
`/var/lib/zookeeper/myid`.

A systemd instance specifier (`%i`) is useful for exactly the kind of thing
that is needed to uniquely identify each Zookeeper instance of the ensemble
(another common usage of the instance specifier seems to be to use the primary
port number a service exposes). This works by splitting the name of the current
service unit's filename on `@` and taking the second part as the instance
specifier value. The recommended way to do this is to create symlinks to your
service file as many times as you want instances of your service, and name them
unit-name@N.service, where N is a number incrementing from 1. Happily, it's
easy to make the symlinks once and then check them into git with the rest of
the configuration. Therefore, if you want 3 instances of Zookeeper (which
happens to be my situation), you might create the symlinks like this:

    # ln -s zookeeper.service zookeeper@1.service
    # ln -s zookeeper.service zookeeper@2.service
    # ln -s zookeeper.service zookeeper@3.service

The only thing left to do is to communicate the
instance specifier to the container, and from there to Zookeeper. The first
part is easily handled with an environment variable argument to the Docker run
command (e.g., `ExecStart=docker run -e ZK_SERVER_NUMBER=%i
emmanuel/zookeeper:3.4.5`). The second part can be handled with a simple
startup script inside the Docker container. The startup script just writes the
value of the environment variable into the aforementioned
`/var/lib/zookeeper/myid` file.

The last step for my basic Zookeeper deployment on CoreOS is a (simple) systemd
service unit file to pull it all together. The main thing here is to pass the
`ZK_SERVER_NUMBER` environment variable to the `docker run` command and to
include the appropriate metadata to instruct fleet to deploy at most one
zookeeper container per host. That, and using the 'instance ID specifier'
(`%i`) and making as many symlinks as you want instances. That looks like:

    [X-Fleet]
    X-Conflicts=zookeeper@*.service

Then you can deal with them as a group with `fleetctl`:

    # fleetctl submit --sign zookeeper@*.service
    # fleetctl start zookeeper@*
    # fleetctl status zookeeper@*
    # fleetctl destroy zookeeper@*

This is what I've put together so far; it adds up to a Zookeeper ensemble (of
arbitrary size) running on top of a CoreOS cluster. Well, except for the part
where I glossed over how the DNS names work. And then, the next step is to make
it visible to other containers. Find out more about both in the next article.

All the code for this article [is available on
Github](https://github.com/emmanuel/docker-zookeeper).
