---
layout: post
title:  "Go makes cross compilation dead simple"
date:   2014-07-17 9:27:43
categories: tech go-lang confd homebrew
---

So, I'm an idiot. [I've been working with docker (and CoreOS)
lately]({% post_url 2014-07-16-working-with-coreos-on-aws %}), which has
involved, among other things, dealing with some projects written in Go (aka go-lang). I've been
interested in Go for a while, but I haven't had a pressing need to learn my way
around it, so I've put off taking the plunge. However, in order to continue
making progress with my CoreOS/AWS/SkyDNS project, I need to be able to
configure services using information with etcd, even when those services are
configured via on-disk files (rather than environment variables or command-line
arguments).

Enter [confd](https://github.com/kelseyhightower/confd). Confd will check (or
poll, or eventually, watch) etcd or consul for service registration info.
Currently, confd supports 2 primary modes: onetime and polling. The onetime
mode simply pull values out of etcd and uses them to build a service's
configuration file from a template. The polling mode is similar, but repeats
the cycle over and over: upon changes to any of a set of watched keys, confd
will rebuild the config file and then run an (optional) command to trigger your
service to re-read its (now updated) configuration.

Let me pause for a moment to point out that, in an ideal (read: fantasy) world,
services would support DNS SRV records natively to discover service ports
dynamically. That is not the world we live in today. If there was widespread
support for SRV records, this article would never have been written, because it
would be totally unnecessary for me to use confd.

Sounds like a perfect fit for my use case. **BUT**, confd does not support JSON
payloads from etcd (yet). Thankfully, there are 2 open pull requests against
confd ([one against the 0.5.x
branch](https://github.com/kelseyhightower/confd/pull/101), and [a second
against the 0.6.x branch](https://github.com/kelseyhightower/confd/pull/102))
which add support for parsing JSON payloads in confd templates. Looks like I
don't have to learn Go just yet. Bullet dodged!

However, I do need to compile this forked version of confd. I work on a
Macbook, so I don't just need to compile confd, I need to cross-compile it for
Linux (specifically amd64). This is where I confess that I'm an idiot, because
I started with the assumption that cross-compilation is hard. So, I started by
spinning up a simple Docker container with a Go toolchain in order to avoid
cross-compiling. That wasn't hard, but boy did it seem like a work-around I
should not need to deal with. Today I learned that my instinct was correct and
my buildbox was a needless distraction.

Here's what it took for me to compile the forked confd:

{% gist emmanuel/f21ddcfb22e3324c8803 %}
