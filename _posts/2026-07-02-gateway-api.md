---
title: "Homelab: The Evolution of Kubernetes Networking - From Ingress to Gateway API"
date: 2026-02-07
categories:
  - homelab
  - kubernetes
tags:
  - nginx
  - gateway-api
  - networking
  - k8s
---

Networking in Kubernetes has always been a journey of trade-offs. Ever since I started working with kuberentes I relied on `VirtualServers` the proprietary NGINX CRDs, and the standard `Ingress` resource.

Now, I've completed the migration[^1] to the new industry standard: the **Gateway API**.

[^1]: I'm still running the other 2 in parallel since I want to further compare them before fully migrating all my apps.

In this post, I’ll share why I started to migrate, compare the three approaches side-by-side using my actual Homelab configurations, and cover the "gotchas" I encountered during installation.

## The Three Stages of Kubernetes Networking

### 1. The Standard Ingress (Simple but Limited)
The `Ingress` resource is the "Hello World" of k8s networking. It's portable, but it lacks depth. To do anything advanced—like rewriting paths, adjusting buffers, or configuring headers—you rely on "Annotation Hell."

```yaml
# The "Old" Way: Annotations everywhere
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
...

```

### 2. The NGINX VirtualServer (Powerful but Proprietary)

To escape annotation hell, one can use the **NGINX VirtualServer**. This is a Custom Resource Definition (CRD) specific to the NGINX Ingress Controller. It treats configuration as structured data, not string annotations.

**Pros:** Extremely powerful. Supports advanced traffic splitting, WAF policies, and precise header manipulation.
**Cons:** Vendor lock-in. If I ever wanted to switch to Istio or Cilium, I'd have to rewrite everything.

This is where the new **Gateway API** shines.

### 3. The Gateway API (The New Standard)

The Gateway API is the successor to Ingress [(which will be deprecated by end of March 2026)](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/). One of its main goals is to provide the means to separate the concerns of "Infrastructure Providers" (e.g. the base infrastructure team), "Cluster Operators" (e.g. the platform team), and "Application Owners" (e.g. developers). Since it is designed as a standard API, there is no vendor lock-in and it can be implemented by any controller (NGINX, Istio, Cilium, etc.).

## Side-by-Side Comparison: Exposing "Immich"

Let's look at how I exposed **Immich** (my photo backup app) using `VirtualServer` vs. the new `Gateway API`.

### The Proprietary Way (VirtualServer)

Notice how this resource mixes *infrastructure* concerns (TLS secrets) with *routing* concerns (upstreams).

```yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: immich-vs
spec:
  host: immich.novotny.live
  tls:
    secret: wildcard-tls  # <--- Repetitive: Defined in every app
  upstreams:
    - name: immich-server
      service: immich-server
      port: 3001
  routes:
    - path: /
      action:
        pass: immich-server

```

### The Modern Way (Gateway API)

Here, the configuration is split. The `Gateway` handles the heavy lifting (TLS, Ports), and the `HTTPRoute` just focuses on the app.

**1. The Gateway (Infrastructure)**
*Defined once by the Platform Team.*

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: homelab-gateway
spec:
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      hostname: "*.novotny.live"
      tls:
        mode: Terminate
        certificateRefs:
          - name: wildcard-tls # <--- Centralized TLS termination!

```

**2. The HTTPRoute (Application)**
*Defined by the App Owner.*

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: immich-http
spec:
  parentRefs:
    - name: homelab-gateway
      sectionName: https # <--- "Plug me into the secure listener"
  hostnames:
    - immich.novotny.live
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: immich-server
          port: 3001

```

## Lessons Learned

### 1. TLS is Finally Centralized

With `Ingress` and `VirtualServer`, I had to ensure the `wildcard-tls` secret existed in every namespace and reference it in every file.
With Gateway API, I reference the certificate **once** in the Gateway. The Routes simply inherit the secure listener. This drastically cleans up my application manifests.

### 2. Installation: The "Chicken and Egg" Problem

Unlike standard Ingress controllers, the Gateway API requires a set of CRDs (the API specification itself) to be installed *before* the controller (NGINX Gateway Fabric), since the API is not built into Kubernetes (as of now). This creates a "chicken and egg" problem during installation.

If you try to `helm install` the controller without them, it fails. I solved this using Kustomize to layer the installation order:

```yaml
# tools/nginx-gateway-fabric/kustomization.yaml
resources:
  # 1. Install the API Standard first (Base Infrastructure Team)
  - https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
  # 2. Then the Controller configuration (Platform Team)
  - gateway.yaml
  
# 3. The Controller's Helm Chart (Base Infrastructure Team)
helmCharts:
  - name: nginx-gateway-fabric
    # ...

```

### 3. Explicit "Attachment"

The `parentRefs` field in the `HTTPRoute` is powerful. By specifying `sectionName: https`, I enforce that this application is **only** accessible over the secure listener. I don't need annotations to force SSL redirects; the architecture itself enforces it.

## Conclusion

Migrating to the Gateway API felt like "growing up" in Kubernetes and removing boilerplate repetition. It offers the best of both worlds:

* **vs Standard Ingress:** It offers the power and expressiveness that standard Ingress lacks.
* **vs VirtualServer:** It offers the portability and standardization that VirtualServer lacks.

I am now running **NGINX Gateway Fabric** alongside my legacy Ingress controller and will explore more advanced features (like traffic splitting and WAF policies) in future posts. But for now, I'm thrilled to have a clean, standardized API for exposing my applications.

