---
layout: post
title: K8s Ingress
summary: A brief look at the different options for exposing services running on Kubernetes using ingress controllers
---

## Too many proxies

I've had a number of questions around ingress controllers for K8s and there is a quite a list. When I talk about ingress, we are really talking about L7 routing and not K8s LoadBalancer. These are a few of the options I found:

* [NGINX](https://github.com/kubernetes/ingress-nginx/blob/master/README.md)
    * This is K8s maintained and the defacto (unless you are on GCE)
* [HAProxy](http://www.haproxy.org/)
* [Envoy](https://www.envoyproxy.io/)
* [Istio](https://istio.io/docs/tasks/traffic-management/ingress.html)
    * Uses Envoy proxy as it ingress controller
* [Ambassador](https://www.getambassador.io/user-guide/getting-started)
    * This also uses Envoy

## Envoy and the service mesh

Ok so Envoy appears three times in the above list, so there might be something to that!

>Envoy is an open source edge and service proxy, designed for cloud-native applications 

That sounds like something that could be useful and as we dig into the docs we find the reason this has become popular with K8s and the other service mesh projects like Istio.

>The network should be transparent to applications. When network and application problems do occur it should be easy to determine the source of the problem.

Even better, working with overlay networks and Docker is never fun and stitching together even small scale microservices can be tough. So Envoy will actually sit alongside out application when it is deployed, intercepting traffic and getting it to the right destination. So we can build our apps and let the service mesh take care of the complex task of figuring out where all our services are.

Service mesh solutions like Istio focused on traffic between microservices, whereas Ambassador is focused on providing an API Gateway for inbound/outbound traffic from the cluster as a whole.

I really like Isito and the concept of the sidecar pattern to take of the networking, but the project we were working on already had the basics in place and just needed external connectivity vs retro fitting the entire deployment with Istio.

So Ambassador was actually very quick and easy to get up and running, we needed the admin service and then the config.

<script src="https://gist.github.com/msimpsonnz/f4c028fc915b30d94671a54f28b471c9.js"></script>

<script src="https://gist.github.com/msimpsonnz/ee68b0ea5cf7ba520106863c3c4a363c.js"></script>

