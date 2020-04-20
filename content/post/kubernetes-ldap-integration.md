---
title: "Kubernetes LDAP Integration via Dex"
date: 2019-12-21T12:06:09Z
draft: true
---
Authenticating into a Kubernetes cluster can be done via: X509 Certs, Static tokens or passwords, service account tokens or with OpenID Connect (OIDC) tokens, see [here](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#authentication-strategies).

In order to authenticate with providers like AD, LDAPs or external providers you can use the OpenID authentication method build into the `kube-apiserver` to talk to an OIDC provider, like [dex](https://github.com/dexidp/dex).

## Notes
This guide was done with:
- Dex `v2.21.0`
- Kubernetes `1.17.0` via kind (via kubeadm)
- Local testing with [kind](https://github.com/kubernetes-sigs/kind) `v0.6.1`
- Gangway `v3.2.0`

# Steps
## 1. Kubernetes
### 1.1 Kubeadm
The [kubeadm config](https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta1) provides a way to configure a cluster with `kubeadm init` and can be used for local testing via _kind_. 

To enable oidc auth the flags are:
```yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
metadata:
  name: config
apiServer:
  extraArgs:
    # The url where the auth providr responds
    oidc-issuer-url: https://auth.example.com/dex
    # Client id as defined in dex config
    oidc-client-id: kubernetes
    # Field from OIDC payload used for username
    oidc-username-claim: email
    # CA cert for issuer url TLS
    oidc-ca-file: /oidc/ca.crt
    # Property for groups entries in OIDC payload
    oidc-groups-claim: groups
    # Prefix for Kubernetes RBAC group resources
    oidc-groups-prefix: "oidc:"
    # Prefix for Kubernetes RBAC user resources
    oidc-username-prefix: "oidc:"
  extraVolumes:
  - name: "ca"
    hostPath: "/cacrts/ca.crt"
    mountPath: "/oidc/ca.crt"
```
Some things to note:

- The `oidc-issuer-url` should return when going to /.well-known/openid-configuration with an OIDC discovery payload
- The `oidc-ca-file` is optional if the cert for the endpoint is already in the cluster certs information but TLS from the apiserver to OIDC is not optional
- The `oidc-{groups,username}-prefix` prevents clashing between LDAP user/group names and ones in the Kubernetes cluster already. It does mean when you reference an LDAP group called "admins" it'll be "oidc:admin" in Kubernetes RBAC resources

## 2. Dex



