---
layout: post
title: 服务发现的几种方式
published: true
---

#Service Discovery Approaches

This summary is taken from the SmartStack [website](http://nerds.airbnb.com/smartstack-service-discovery-cloud/) 

###DNS

**Cons:**

1. poll-only, no push

2. propagation delays is non-deterministic, because of 
various layers of caching in the DNS infrastructure

3. only support random routing(load balancing)

###Central load balancer

**Cons:**

1. single-point of failure

2. increased network hop(roundtrip)

###In-app registration/discovery

**Cons:**

1. different languages need different client library

2. need to inject code to application




