---
title: "Kubernetes Networking #3: Ingress"
date: 2022-07-22T18:06:09Z
draft: false
categories:
- Deep dive
---

Kubernetes Ingress provide a way to expose HTTP(S) routes into your pods from a centralized controller and load balancer. It means you can define *how* the network traffic can get to your pod inside a Kubernetes resource and a controller hosted in the cluster takes care of the implementation.

<!--more-->

In a [world without Kubernetes](https://en.wikipedia.org/wiki/Utopia), you might have a reverse proxy deployed where you write the configuration file to detail how to expose your applications in the backend. An ingress controller takes that reverse proxy and automatically generates the config based on the `Ingress` Kubernetes resources you create. You don’t have to create a new `LoadBalancer` service and cloud load balancer for each service you want to expose.

In this guide, we’ll use the Kubernetes maintained ingress controller, [ingress-nginx](https://github.com/kubernetes/ingress-nginx), which uses nginx as it’s reverse proxy. It’s important to note, this isn’t the [nginx-ingress-controller](https://docs.nginx.com/nginx-ingress-controller/) from NGINX themselves which has hooks into NGINX plus and other paid offerings. The docs for ingress-nginx can be found [here](https://kubernetes.github.io/ingress-nginx/).

## 🤷 Why not just LoadBalancer?

Kubernetes provides a way for a cloud provisioned load balancer (an [AWS elastic load balancer](https://aws.amazon.com/elasticloadbalancing/) for instance) to be deployed by the control plane, see my [services post](https://dgood.win/post/kubernetes-networking-2-services/) for more info. We can use these to expose a single service (and collection of pods) from a load balancer. But what happens when you have 50 services deployed onto your cluster? Would you have 50 load balancers for them all? That’s ~$810 in standing charges every month not including paying for the traffic that goes through them.

This is where an ingress controller comes in. An Ingress controller is a centralized place for traffic to enter your cluster via a single load balancer which then is routed to your backend services automatically. This is all based on `Ingress` resources deployed on the cluster, through the reverse proxy.
It also introduces a place to add some control over your traffic routing. kube-proxy will default (when in `iptables` mode) to TCP load balancing traffic randomly, the traffic coming in via a load balancer goes to any pods. With the controls you can configure affinities, round-robin routing, rate limits, CORS etc. to really shape the traffic going to your pods centrally.

These diagrams compare the path a request would take with/without an ingress controller exposing your pod:

{{< figure src="/img/ingress-lb-to-pod.png" title="Without: Application Service LoadBalancer → NodePort → Randomly to one of your pods" caption="Without: Application Service LoadBalancer → NodePort → Randomly to one of your pods" >}}

{{< figure src="/img/ingress-controller-to-pod.png" title="With: Ingress-controller service LoadBalancer → NodePort → Randomly to an ingress-controller pod → To one of your application pods via ClusterIP" caption="With: Ingress-controller service LoadBalancer → NodePort → Randomly to an ingress-controller pod → To one of your application pods via ClusterIP" >}}

There are some tradeoffs to using an ingress controller:

- This introduces an extra hop for your network requests to reach your pod. If you have high throughput - and low latency is important - this could be a consideration
- In the same vein, the ingress controller pods also use resources. If you have the ingress controller doing a lot of TLS termination or handling high load the resource usage will increase significantly
- You can only really use this for (mostly) Layer 7 traffic, TCP or UDP cannot directly be exposed this way

## 🛠️ So what does an ingress controller actually do?

At a high level you have the following components:

- **Kubernetes resources:**
    - Service: type `LoadBalancer`, meaning your ingress-controller pods are fronted by a cloud load balancer and exposed outside the cluster
    - Deployment: Ingress Controller itself (see components below for a breakdown)
    - (sometimes)Deployment: default backend, this is where the ingress controller routes traffic to when there isn’t a matching host/path rule for the traffic coming in
- **Components of the ingress controller:**
    - Reverse proxy: This is what actually routes the traffic coming in to downstream services/pods. If you installed nginx on a service and ran it it’s this basically
    - Config generator: This continuously gets the state of Ingress resources in the Kubernetes cluster (think of it as spamming `kubectl get ingress -A` every X-seconds) and uses that to generate config that the reverse proxy (nginx in this case) understands

The *reverse proxy* continually has it’s config updated by the *config generator* and then traffic coming into the cluster via the *Loadbalancer service* hits the reverse proxy and is routed to the proper backend service/pods.

## 🔒 What about SSL?

For HTTPS traffic, we can configure the ingress-controller to terminate the SSL and provide the certs as needed.

When creating an ingress resource you can specify a Kubernetes Secret of type `tls` and the ingress-controller will use that secret to terminate the SSL of the request at the reverse proxy.

This could also be coupled with a tool like [cert-manager](https://cert-manager.io/docs/) to dynamically issue and renew certificates for you.

## 🎛️ Load balancer/Reverse proxy controls

As mentioned before, by having your pods traffic route via a reverse proxy in the cluster you gain more control over the routing and connection. Some useful examples for ingress-nginx are:

- **Authentication**: Setup [basic or digest access authentication](https://datatracker.ietf.org/doc/html/rfc2617) on routes
    - `nginx.ingress.kubernetes.io/auth-type: basic`
- **Session affinity:** Allows for things like sticky sessions, keeping a user going to the same backend pod or use cookie affinity to apply a `SameSite` sticky cookie
    - `nginx.ingress.kubernetes.io/affinity: sticky`
- **SSL passthrough:** Instead of terminating TLS at the load balancer you can let the backend handle it directly
    - `nginx.ingress.kubernetes.io/ssl-passthrough: "true"`
- **Redirections:** Both [permanent](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#permanent-redirect) and [temporary](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#temporal-redirect)
    - `nginx.ingress.kubernetes.io/permanent-redirect: https://www.google.com`
- **Security controls**
    - [SSL ciphers](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#ssl-ciphers) can be configured controlling which SSL ciphers you support
        - `nginx.ingress.kubernetes.io/ssl-ciphers: "ALL:!aNULL:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP"`
    - [CORS rules](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#enable-cors) to configure the cross-original resource sharing rules
        - `nginx.ingress.kubernetes.io/enable-cors: "true"`

A lot of these controls can be configured either as [annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/) directly on Ingress resources affecting only those specific routes, or can be provided in the [Configmap](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/) to be applied at a global level.

## 🤹‍♂️ Can I have multiple ingress-controllers?

**Yes**. Let’s say you want to be able to accept traffic from the public internet as well as internally from inside your private network.

We would:

- Deploy two sets of ingress-controllers, *ingress-public* and *ingress-private*
- Configure them to be public and private. This is done usually by the annotations given to the different `Service` resources (`service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"` for instance)
- Setup the Ingress classes
    - This is a relatively new resource, it used to be managed by an annotation `kubernetes.io/ingress.class` which is being deprecated
- When creating `Ingress` resources you can target the `IngressClass` to specify which controller should setup routing for your traffic

## 📮 What about hostnames?

Ingress resources in Kubernetes allow you to specify both the Paths and the Host values to route traffic for.

To have your ingress controller exposed via a friendly DNS address you’d need to point a record of one to your load balancers DNS address. In AWS, this would be creating a Route53 A record aliased to your ELBs address. This would mean anything going to `friendly-address.domain.com` would point to your cloud-provisioned ELB and then onto your reverse proxy pods from there.

Then any routes off this hostname would be directed to your specified pods, ie.:

```
`friendly-address.domain.com/app-a` → `svc/app-a` in the cluster
`friendly-address.domain.com/app-b` → `svc/app-b` in the cluster and so on
```

But what if you wanted to have multiple different hostnames, maybe 1 per application.

```
`app-a.friendly-address.domain.com/` → `svc/app-a`
`app-b.friendly-address.domain.com/` → `svc/app-b`
```
These would mean adding a new Route53 entry manually every time you want to deploy a new application. It takes away from the benefits of defining Ingress as a Kubernetes resource if you need to go click-ops or Terraform a new host each time.

That’s where a tool like [external-dns](https://github.com/kubernetes-sigs/external-dns) comes in.

### external-dns

external-dns allows you to automatically configure your DNS services based on your Kubernetes Ingress resources (and services!).

As an example, if you have deployed external-dns and wanted to setup a new `app-c.friendly-address.domain.com` to point to your service all you would need to do is create an `Ingress` resource with the `Host` field set to `app-c.friendly-address.domain.com` and external-dns would provision you an A record pointing to your load balancer.

## 🍵 Bringing it all together

To give an example of all of this in use, lets build resources for the following requirements:

- We have 3 applications: `alpha` `beta` and `gamma` running on our cluster
- We want `alpha` and `beta` exposed via the same Hostname but with separate routes:
    - `public.example.com/alpha` and `public.example.com/beta`
- We want `gamma` to have it’s own hostname as it’s a separate, unrelated application
    - `gamma.public.example.com`
- We have the TLS key and cert for `public.example.com` and want the ingress-controller to terminate TLS for this host
- `gamma` handles it’s own TLS so we shouldn’t terminate it at the reverse proxy,

We already have:

- A Kubernetes cluster running in AWS
- Deployed external-dns onto our cluster configured for AWS Route 53, [[link](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md)]
- Deployed ingress-nginx onto our cluster configured for AWS, [[link](https://kubernetes.github.io/ingress-nginx/deploy/)]. Ingress class is set to `default`

Feel free to try these steps above for yourself the implementation and breakdown of what we’re doing come below though so spoiler alert.

---

1. Create the TLS secret for our alpha/beta endpoint
    
    ```
    kubectl create secret tls public \
      --cert=tls.cert \
      --key=tls.key
    ```
    
2. YAML for the Ingress of `alpha` and `beta`
    
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: alpha-beta
    spec:
      ingressClassName: default
      tls:
      - hosts:
          - public.example.com
        secretName: public
      rules:
      - host: public.example.com
        http:
          paths:
          - path: /alpha
            pathType: Prefix
            backend:
              service:
                name: alpha
                port:
                  number: 8080
          - path: /beta
            pathType: Prefix
            backend:
              service:
                name: beta
                port:
                  number: 8080
    ```
    
3. YAML for the Ingress of `gamma`
    
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: gamma
      annotations:
        nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    spec:
      ingressClassName: default
      rules:
      - host: gamma.public.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: gamma
                port:
                  number: 8080
    ```
    
> ☝ SSL Passthrough also needs to be enabled on the ingress-nginx deployment with the flag `--enable-ssl-passthrough`
        

**What happens with these resources?**

- **ingress-nginx:**
    - Constantly checks for new/updated `Ingress` resources
    - Detects `alpha-beta`:
        - Adds nginx config to route to the services
            - Can be checked with: `kubectl exec -it <ingress-controller-pod-name> -- cat /etc/nginx/nginx.conf`
            - This config includes the secret mentioned to be able to terminate TLS
        - Updates the `Ingress` resource `Address` field with the DNS of the Cloud load balancer the controller knows is pointing at it
        - `curl https://public.example.com/alpha -v` shows it routing correctly and the cert as expected
        - `curl https://public.example.com/wrong -v` shows the default backend
    - Detects `gamma`:
        - Adds nginx config to route to the services
            - Again, can be checked with: `kubectl exec -it <ingress-controller-pod-name> -- cat /etc/nginx/nginx.conf`
            - This config will show the ssl_passthrough is set to not handle any TLS
        - Updates the `Ingress` resource `Address` field with the DNS of the Cloud load balancer that the controller knows is pointing at it.
        - `curl https://gamma.public.example.com -v` shows traffic routing correctly and with our in-application TLS handling
- **external-dns:**
    - Constantly checking for new/updated `Ingress` resources
    - Sees the two new resources but with empty `Address` fields - needs these to be set before it can point the specified host to the address the service sits behind.
    - `Address` field is now updated for `alpha-beta`:
        - Checks if the hostname has already been created
            - external-dns defaults to using `txt` records in the chosen DNS service to persist information about the domains it manages
        - Creates an `A` record pointing `public.example.com` to the value of `Address` in the ingress
    - `Address` field is now updated for `gamma`
        - This does the same as `alpha-beta` but for `gamma.public.example.com`

## 🦦 Other controllers are available

Other ingress controllers are available too, notable/interesting ones include:

- [AWS Load balancer controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller#readme) can manage ALBs out of cluster for ingress resources and NLBs for service resources
- [Traefix ingress controller](https://doc.traefik.io/traefik/providers/kubernetes-ingress/) an ingress controller using the Kubernetes native edge router
- [Ambassador](https://www.getambassador.io/docs/edge-stack/latest/topics/running/ingress-controller/) uses Envoy to act as an ingress controller and API gateway

The benefit of having all of your configuration directly in Kubernetes as an `Ingress` resource comes from every compliant ingress-controllers reading and interacting with Ingress in the same way. The only change between them would be the implementation specific things you control via annotations, the general routes and hosts work regardless of your choice.