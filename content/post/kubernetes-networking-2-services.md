---
title: "Kubernetes Networking #2: Services"
date: 2022-04-21T10:06:09Z
draft: false
categories:
- Deep dive
---

Kubernetes services are an essential resource everyone uses, but what are all the different types and how do they work under the hood?

<!--more-->

In Kubernetes pods are ephemeral (or [cattle](https://thenewstack.io/how-to-treat-your-kubernetes-clusters-like-cattle-not-pets/)), you should expect them to be killed, scaled and replaced whenever. So how do you talk to a set of pods which might have changed the next second? A service.

Services sit in front of your application and mean you only have to call the service address and magic Kubernetes components will get that request to the right Pod(s). 
How does a service know which pods it can talk to?

## Selectors

Services map to pods via label selectors. You have a pod with label `app=etizer` and then create a service with `.spec.selector` for the same and that service will forward traffic to any Ready pods matching that label.

Behind the scenes, a service resource also creates an `Endpoints` resource which maintains a list of Pod IPs based on your service selector. It’s worth noting here, the readyiness checks you implement for your Pods are what controls whether the IP goes into this `Endpoints` resource and allow traffic.

The exception here is selector-less services, these don’t have a label selector so won’t also have an `Endpoint` resource. To use these you also need to create an endpoint resource of the same name as your service, you can add any IPs and ports you want to this resource. This is a great way to map a service to something not in your cluster so your pods can talk to it as if it was a service.

But what are the different types of services and how do they work?

## Service Types

### ClusterIP

When you create a `ClusterIP` type service you’ll will get:

- A virtual IP for the service
    - This is within the `--service-cluster-ip-range` your `api-server` has been configured with
    - `kube-proxy` is responsible for this virtual IP routing to one of your Pod IPs
    - This virtual IP will forward requests to one of your pod IPs based on the `Endpoints` resource associated with your service.
- A local cluster DNS address, eg. `<service-name>.<namespace>.svc.cluster.local`
    - This is resolved by your `kube-dns` (CoreDNS) to your virtual service IP
    - This is why your service name must be [RFC 1035 compliant](https://datatracker.ietf.org/doc/html/rfc1035)

The normal way of utilising this would be giving your application the service DNS address so it can talk to another application running on the cluster. This DNS address resolves to the virtual IP and then the configuration `kube-proxy` controls handles that IP resolving to actual pod IP addresses. From there your CNI will route your request using the pod IP. 

The `ClusterIP` service type is ✨*special*✨, whilst you use it as a way to expose your service internally within the cluster, it’s also used by types `NodePort` and `LoadBalancer` (by extension of LoadBalancer using `NodePort`) as part of their implementations. 


{{< figure src="/img/k8s-services-2-types.png" title="Just like a Matryoshka doll" caption="Just like a Matryoshka doll" >}}


`<service-ip>:<service-port> → <pod-ip>:<pod-port>`

### NodePort

A `NodePort` type service opens up a port on **every node** in your cluster, this port then proxies requests to your service’s virtual IP address.

The port range defaults to `30000-32767`, interestingly enough just below the Linux kernel default [ephemeral port range](https://www.kernel.org/doc/html/latest//networking/ip-sysctl.html#ip-variables). It can also be configured with the flag `--service-node-port-range` if you need these ports for some use case.

At a high level a `NodePort` is a single rule configured by `kube-proxy` onto every node, to forward requests to an underlying `ClusterIP` service.

`<node-ip>:<node-port> → <service-ip>:<service-port> → <pod-ip>:<pod-port>`

### LoadBalancer

A `LoadBalancer` type service only really works in a cloud environment. When created, asyncronously, the [cloud-controller-manager](https://kubernetes.io/docs/concepts/architecture/cloud-controller/) will provision you a load balancer which directs traffic to your pods.

The implementation of the different cloud load balancers varys wildly. It used to be different clouds were implemented [in-tree](https://github.com/kubernetes/kubernetes/tree/release-1.11/pkg/cloudprovider/providers/aws), committing directly to kubernetes but now it must be done via out-of-tree implementations, eg. [cloud-provider-aws](https://github.com/kubernetes/cloud-provider-aws).

Cloud implementations aside, once you have a *cloud* load balancer it works by pointing to a `NodePort` service running. This means that you have a load balancer with every Kubernetes worker node as a listener, forwarding traffic to any nodes on a particular port. From there it follows the standard `NodePort` → `ClusterIP` route.

If you didn’t want to write a whole [cloud control manager](https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/) to expose your application on your own servers, a `NodePort` service with that port pointed to from your existing load balancer would do the trick.

`<load-balancer-ip>:<lb-port> -> <node-ip>:<node-port> → <service-ip>:service-port> → <pod-ip>:<pod-port>`

### ExternalName

`ExternalName` services are the odd one out. Instead of using good ol’ `ClusterIP` and getting a virtual IP, they create a `CNAME` record to map your service DNS to another DNS address.

As an example use case, lets say you have different environment databases, instead of service-owners changing the DNS depending on where they deploy their pods to you create a `ExternalName` service:

```bash
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: prod
spec:
  type: ExternalName
  externalName: prod.database.example.com
---
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: dev
spec:
  type: ExternalName
  externalName: dev.database.example.com
```

Now all the service-owners have to do is point their code to `database.svc.cluster.local` and they will talk to the right database every time.

`database.svc.cluster.local` = `CNAME test.database.example.com`

It’s worth noting since the pod requesting the service will get the `CNAME` record back, the pod also has to be configured to resolve whatever domain the `CNAME` returns. If it’s a private record, you might want to [configure revolvers in CoreDNS](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/).

## How Virtual IPs work

*Most* services (sorry, `ExternalName`) use a virtual IP to forward requests to the proper pods. It’s the responsibility of `kube-proxy` to provide this virtual IP and make sure the traffic is actually forwarded. `kube-proxy` runs on every node so the steps below happen per worker node.

How this works depends on the mode of `kube-proxy` is given with `-proxy-mode` flag:


> ℹ️ **For all of these `kube-proxy` will monitor the control plane for new/updated/removed `Service` and `Endpoint` resources and run the following steps**


- **userspace** (legacy)
    - For each `Service` it opens a random proxy port on the node
    - This proxy port is then told to iptables to capture traffic to this particular virtual IP and port to redirect to the Pod IPs in the `Endpoint` resource
    - The backend pod is chosen based on the `SessionAffinity` of the service
    - **The traffic here is actually proxied via `kube-proxy` itself**
- **iptables**
    - For each `Service` it creates iptables rules which redirect traffic to the virtual IP and service port
    - These redirect rules point to sets of pod IPs based on what is in the endpoint resource
    - The backend pod is chosen at random by default
    - **The traffic is all handled within the Linux [netfilter](https://www.netfilter.org/) so all within the kernelspace, `kube-proxy` doesn’t handle the traffic it just sets up rules to process it**
- **IPVS**
    - Whilst iptables was designed to be a firewall and `kube-proxy` just uses it to redirect network traffic, IPVS was made for load balancing
    - At a high level this works similar to iptables, cluster IP and port → set of pod IPs
    - IPVS is important when working at scale, if you have >1000 services (or 10,000 pods) in your cluster IPVS is the performant choice
- **kernelspace** (windows)

## Bringing it all together

Let’s create an example `LoadBalancer` service and see what components come together to get traffic from a client to your pods. We’ll show how traffic from outside, via the load balancer, gets to your pods and how another pod in the cluster can also use this service without going via the load balancer.

In this example let’s say we’re running on a Kubernetes cluster in AWS with the AWS cloud controller manager properly configured, we have pods running with label `app=alpha` which listens on port `8080` for traffic and `kube-proxy` is in the default `iptables` mode.

- Service YAML
    
    ```bash
    apiVersion: v1
    kind: Service
    metadata:
    # This doesn't affect routing but it's best practice to label components together
      labels:
        app: alpha
      name: example
    spec:
      ports:
      - name: main
        port: 8081
        protocol: TCP
    # Target port is the port your pod has open
        targetPort: 8080
    # This is what tells the service which pods to route to
      selector:
        app: alpha
      type: LoadBalancer
    ```

Lets create the service and have a look:

```bash
$ kubectl apply -f service.yaml
service/example created

$ kubectl get service example
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
example   LoadBalancer   10.43.205.153   192.168.0.147   8081:32255/TCP   4s
```

I now have:

- An Endpoints resource corresponding to ready pods matching my label selector
    - `kubectl get endpoints example`
- A cluster IP setup for my service `10.43.205.153`
- A DNS address for the service configured
    - (from inside a Pod)
    - `dig example.default.svc.cluster.local`
- A external (to the cluster) IP which maps to my load balancer `192.168.0.147`
- A NodePort setup `32255` so all my workers now have this port open
    - `netstat -tulnp | grep 32255`
- iptables rules setup
    - `sudo iptables -L | grep example`
    - `sudo iptables -L | grep 10.43.205.153`

Lets see how this request goes through to our pod from outside the cluster and a pod within.

### From outside the cluster


{{< figurelink src="/img/k8s-services-2-lb-external.png" title="External from cluster Loadbalancer to pod" link="/img/k8s-services-2-lb-external.png" >}}


1. The client makes a request to the load balancer to try and talk to the application, for how this is configured see my post on ingress into a cluster
2. The load balancer selects one of the configured listeners, this is usually either random or round robin but depends on your load balancer and configuration
3. The node has port 32255 open, requests here are configured in iptables to forward the request to the virtual IP of the service. These rules were configured by `kube-proxy`
4. iptables then resolves the virtual/service IP into a Pod IP address, one is chosen by random and the request heads there. Now the traffic is inside the cluster the CNI controls the network movements between nodes (see [here](https://dgood.win/post/kubernetes-networking-overview/) for more info on what the CNI sets up)
5. Once the request has reached the node with the pod on it, the CNI and iptables will forward it to the Pod as expected

### From a pod within the cluster


{{< figurelink src="/img/k8s-services-2-lb-internal.png" title="Internal from pod to pod" link="/img/k8s-services-2-lb-internal.png" >}}

1. The requesting pod will have been configured with the service DNS address `example.default.svc.cluster.local`. As it makes this request from within the cluster CoreDNS running will be the DNS resolver. This request will be resolved to the Service (virtual) IP address: `10.43.205.153`
2. Making a request to this address will hit the rules in iptables, configured by `kube-proxy`, this resolves the service IP into a Pod IP by randomly selecting a pod
3. The rest of the request continues as in the external example.
