---
title: 'Building a Kubernetes Certificate Signer Operator: Lessons Learned'
date: 2026-02-03
permalink: /posts/2026/02/03/building-k8s-signer-operator/
tags:
  - kubernetes
  - operator
  - go
  - security
  - testing
---

# Building a Kubernetes Certificate Signer Operator: Lessons Learned

In this post, I want to share my experience building `signer`, a Kubernetes controller that implements a **Signer** for `PodCertificateRequest` (PCR) resources. While the project itself was a fascinating deep dive into Kubernetes internals and the `certificates.k8s.io` API, the process of building it taught me invaluable lessons about operator development, testing pipelines, and obscure API server configurations.

You can find the project on GitHub here: [novog93/signer](https://github.com/novog93/signer).

---
DISCLAIMER: This project is only a toy implementation for my education and experimentation purposes. Do NOT use in production environments.
If you are looking for a production-ready solution, consider using [cert-manager](https://cert-manager.io/docs/usage/kube-csr/) or other established tools.
---

---

## What is Signer?

At its core, `signer` is a controller that watches for `PodCertificateRequest` resources and automatically issues X.509 certificates for pods using a self-signed Certificate Authority (CA). It allows workloads to request identity documents dynamically using the Projected Volume mechanism, making mTLS between pods significantly easier to bootstrap in a lab environment.

But this post isn't just about what the tool *does*—it's about how I built it and the specific hurdles I overcame.

## Key Learnings

Developing the Kubernetes Operator was 20% writing business logic and 80% figuring out how to test and deploy it effectively. Here are my main takeaways from this journey—specifically the things I wish I had done differently from day one.

### 1. Don't Rely Solely on Unit Tests

In the beginning, I relied too heavily on unit tests to validate my logic. By mocking the Kubernetes client in Go, I could verify the certificate generation logic (handling RSA vs ECDSA, validity periods, etc.) instantly.

However, **I wish I had implemented End-to-End (E2E) tests much earlier.**

While unit tests are excellent for business logic, they completely miss the nuances of the actual Kubernetes API behavior. I spent too much time iterating on code that "passed tests" but failed immediately when trying to update a real `PodCertificateRequest` status or handle a conflict. If I were to start over, I would prioritize a functional E2E loop alongside my first unit tests, rather than treating it as an afterthought.

### 2. Escape the "CD Pipeline" Trap with Kind

My biggest mistake was initially relying on a slow, remote feedback loop. My workflow looked like this:

1. Write code.
2. Commit and push.
3. Wait for the GitHub Actions CD pipeline to build the container image.
4. Deploy that image to [my `kubeadm` cluster](/posts/2025/08/20/homelab-part1/).
5. Debug.

This cycle was painfully slow. **I wish I had set up a local Kind (Kubernetes in Docker) environment earlier.**

Once I finally shifted to a local setup, my productivity skyrocketed. I wrote a [shell script](https://github.com/NovoG93/signer/blob/main/scripts/e2e-test.sh) (`scripts/e2e-test.sh`)[^1] that automates the entire lifecycle locally:
[^1]: The script is heavily inspired by the [e2e test of external-dns](https://github.com/kubernetes-sigs/external-dns/blob/master/scripts/e2e-test.sh).

1. **Spins up a local cluster** with the required kubeadm config using [Kind](https://kind.sigs.k8s.io/).
2. **Builds the operator** instantly using `ko`[^2] (bypassing the need for a registry).
3. **Deploys the operator** using Kustomize/Helm.
4. **Verifies behavior** by launching a [test pod](https://github.com/NovoG93/homelab/blob/2d0a3a8de58e495848c73c404766be0dd9859b9f/apps/netshoot/netshoot.yaml) (`netshoot`) requesting a certificate.
[^2]: [ko](https://github.com/ko-build/ko)

Moving away from the "commit-and-wait" loop to a "run-script-and-verify" loop definitely saved me some time waiting on CI runners.

### 3. Configuring Kind Correctly

Using **Kind** for testing is standard practice, but it requires specific configuration when dealing with alpha/beta features.

Since my operator relies on the `PodCertificateRequest` API, simply enabling the feature gate wasn't enough. I learned the hard way that you must ensure **both** the `--feature-gates` and the `--runtime-config` are set.

If you only set the feature gate, the API server might recognize the feature flag but still refuse to serve the API endpoint.

Here is the snippet from my Kind configuration that finally got it working:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
featureGates:
  PodCertificateRequest: true
runtimeConfig:
  "certificates.k8s.io/v1beta1/podcertificaterequests": "true"
```

Without that `runtimeConfig` line, `kubectl get podcertificaterequests` would return a "resource type not found" error, even though the feature gate was technically true.

## Conclusion

Building `signer` was a great exercise in understanding the `certificates.k8s.io` API group and enabling features with kubeadm. By prioritizing a robust local testing pipeline and understanding the nuances of API server configuration, I was able to iterate much faster—once I stopped waiting for my CD pipeline to do the work for me.

If you are interested in how the controller works under the hood or want to see the full E2E script in action, check out the [repository](https://github.com/novog93/signer).
