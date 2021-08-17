---
title: "Local testing Kubernetes cluster with Kind"
date: 2021-04-17T07:59:00+02:00
---

Should you not already have a nice cluster to test on, you can spin up a local Kubernetes cluster using Kind. Kind uses Docker containers to represent nodes. Unlike minikube, this one boots up in less than a minute and can easily be extended with additional pretend nodes, making it superior for working with distributed applications.

There are several ways to install kind. If you are a Go developer, the easiest way may be to 
```bash
GO111MODULE="on" go get sigs.k8s.io/kind@v0.10.0
```
Alternatively, since it is a Go static binary, you can download it from [Kind's Github release page](https://github.com/kubernetes-sigs/kind/releases/tag/v0.10.0).

Here is a good starting configuration for your test cluster:

{{% code file="/static/install-kind/kind-cluster.yaml" language="yaml" %}}

Build a cluster from this definition:
```bash
kind create cluster --config kind-cluster.yaml
```

We should now have a local Kubernetes cluster. The command above updates the kubectl config file and sets itself as current context, so at this point, it should be possible to do this:
```bash
kubectl get pods -A
```

The pod list looks something like this:
```
NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
kube-system          coredns-74ff55c5b-m9tmn                      1/1     Running   0          100s
kube-system          coredns-74ff55c5b-s2kkb                      1/1     Running   0          100s
kube-system          etcd-kind-control-plane                      1/1     Running   0          107s
kube-system          kindnet-btfx8                                1/1     Running   0          86s
kube-system          kindnet-cq9hn                                1/1     Running   0          100s
kube-system          kindnet-dzhlz                                1/1     Running   0          86s
kube-system          kindnet-h6dwx                                1/1     Running   1          86s
kube-system          kube-apiserver-kind-control-plane            1/1     Running   0          107s
kube-system          kube-controller-manager-kind-control-plane   1/1     Running   0          107s
kube-system          kube-proxy-6ttn4                             1/1     Running   0          86s
kube-system          kube-proxy-7dcl5                             1/1     Running   0          100s
kube-system          kube-proxy-r55lq                             1/1     Running   0          86s
kube-system          kube-proxy-rwtf7                             1/1     Running   0          86s
kube-system          kube-scheduler-kind-control-plane            1/1     Running   0          107s
local-path-storage   local-path-provisioner-78776bfc44-blqdz      1/1     Running   0          100s
```

At Docker level, it looks like this:
```
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS                                           NAMES
79c77b7c6bb7        kindest/node:v1.20.2   "/usr/local/bin/entr…"   6 minutes ago       Up 6 minutes                                                        kind-worker2
85266fde34c0        kindest/node:v1.20.2   "/usr/local/bin/entr…"   6 minutes ago       Up 6 minutes                                                        kind-worker3
83d5e58746a3        kindest/node:v1.20.2   "/usr/local/bin/entr…"   6 minutes ago       Up 6 minutes                                                        kind-worker
0f3772edeb62        kindest/node:v1.20.2   "/usr/local/bin/entr…"   6 minutes ago       Up 6 minutes        0.0.0.0:80->80/tcp, 127.0.0.1:32941->6443/tcp   kind-control-plane
```

In order to be able to expose services to the outside world (i.e. to you as developer), we need an ingress controller. Let's get [Ambdassador](https://www.getambassador.io/). Ingress controllers tend to require custom annotations to do what you want, but this gives us some ability to verify our manifests during development.

```bash
kubectl apply -f https://github.com/datawire/ambassador-operator/releases/latest/download/ambassador-operator-crds.yaml
kubectl apply -n ambassador -f https://github.com/datawire/ambassador-operator/releases/latest/download/ambassador-operator-kind.yaml
```

## Metrics server

Kind does not come with a metrics-server, but since it is included in e.g. AKS, some manifests assume it is installed. Let's install metrics-server manually.
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.4.2/components.yaml
```

As noted in [Resource Metrics API (metrics-server) #398](https://github.com/kubernetes-sigs/kind/issues/398), we need to patch the resulting deployment to not verify TLS certificates.
```bash
kubectl patch deployment metrics-server -n kube-system -p '{"spec":{"template":{"spec":{"containers":[{"name":"metrics-server","args":["--cert-dir=/tmp", "--secure-port=4443", "--kubelet-insecure-tls","--kubelet-preferred-address-types=InternalIP"]}]}}}}'
```

## Running a local registry

In order to test your service in Kubernetes, you need to deploy images before they are pushed to your container registry for deployment. This requires running a local registry that is available to your Kubernetes. Starting a registry is simple:

```bash
docker run -d -p 127.0.0.1:5000:5000 --name kind-registry registry
docker network connect kind kind-registry
```