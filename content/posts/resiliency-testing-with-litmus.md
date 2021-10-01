---
title: "Resiliency Testing With Litmus"
date: 2021-05-13T20:54:10+02:00
draft: true
summary: |
    This blog post builds on previous posts about Tekton to demonstrate how we can extend a pipeline to include [chaos engineering](https://principlesofchaos.org/) techniques in order to validate the integrity of a distributed system in its natural habitat.
---
## Background

Chaos Engineering techniques are typically wielded by ops or SRE teams in order to "identify weaknesses before they manifest in system-wide, aberrant behaviors" in QA or production environments, that is against a full system with a live or lifelike state. 

However, in the spirit of DevOps, we should not postpone to operation what we can handle in development. Indeed, chaos engineering techniques can also be applied as part of the automated testing of a part of the system, thus blocking deployment when it displays "aberrant behaviors" under stress.

Enter [Litmus](https://litmuschaos.io/) which allows us to formulate hypotheses about a system running on Kubernetes, inject faults into the system and verify those hypotheses. Litmus has recently reached version 2.0 which addresses some of the previous short-comings and introduces a snazzy user interface.

Olric is a TCP-based service and will therefore require a "layer 2" load balancer. If you are following along these blog posts, you may be using Kind. Follow the [Kind Loadbalancer instructions](https://kind.sigs.k8s.io/docs/user/loadbalancer/) to set up a proper load balancer (trying to use NodePort directly turned out to be a proper rabbit hole because of Docker networking).

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/master/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/master/manifests/metallb.yaml
kubectl apply -f metallb-config.yaml

kubectl apply -f static/resiliency-testing-with-litmus/olric.yaml

## Installing the Litmus controller

Litmus supports multi-tenancy and its controller lives in a special namespace so as to limit access to the controller.

```bash
kubectl create ns litmus
kubectl apply -f https://litmuschaos.github.io/litmus/2.0.0/litmus-2.0.0.yaml
```

kubectl apply -f https://hub.litmuschaos.io/api/chaos/2.0.0?file=charts/generic/experiments.yaml