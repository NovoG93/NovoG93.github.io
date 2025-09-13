---
title: 'Setting up a Kubernetes Cluster the Hard Way with kubeadm (GitOps Series, Part 1)'
date: 2025-08-20
permalink: /posts/2025/08/20/homelab-part1/
tags:
    - kubernetes
    - gitops
    - kubeadm
    - homelab
    - devops
    - cloud-native
---

# Setting up a Kubernetes Cluster the Hard Way with kubeadm (GitOps Series, Part 2)

In the previous post of this series, we set up a Kubernetes cluster from scratch using kubeadm. Now that we have a functional cluster, it's time to deploy some essential infrastructure applications and tools to manage our cluster effectively.

Hence this post will now desribe how to:

1. Setting up core infrastructure components and tools
   0. Adding taints and labels to the nodes
   1. Core
      1. Calico as CNI plugin via tigera-operator
      2. MetalLB as load balancer
      3. Metric-server for resource metrics
   2. Tools
      1. ArgoCD for GitOps
      2. Vault + External Secrets Operator for managing secrets
      3. Enabling NFS for persistent storage
      4. Configuring cert-manager for managing TLS certificates
      5. Deploying nginx-ingress controller for ingress management
      6. Configuring kyverno to populate ingress resources with TLS certificates
      7. Installing pi-hole for DNS management and ad-blocking
      8. Configuring external-dns for dynamic DNS updates via pi-hole

# Part 2: Setting up core infrastructure components and tools

## Prerequisites

Before we begin, ensure you have followed [part 1 of this series](https://novog93.github.io/posts/2025/08/20/homelab-part1/) to set up your Kubernetes cluster using kubeadm. You should have a functional cluster with at least one master and two worker node.
Further, create a git repository to store your configuration files. In this and the following tutorial I will guide you to utilize a repository with a folder structure as shown below:

```shell
.
├── apps
├── core
├── helmCharts
└── tools
```

> I pin chart versions below. Feel free to bump later—just keep them pinned for reproducibility.

## Setting up core infrastructure components and tools

### Adding taints and labels to the nodes

For my homelab setup I want to be able to determine which workloads run on which nodes. Therefore I will add some taints and labels to the nodes.

```bash
kubectl taint nodes wp1 tools=true:NoSchedule
kubectl label nodes wp1 tools=true
kubectl taint nodes wp2 apps=true:NoSchedule
kubectl label nodes wp2 apps=true
```

With this configuration, the node `wp1` is designated for running tools and will not schedule any other workloads, while `wp2` is designated for running applications.

### Core

Now we will setup core infrastructure components with the help of [kustomize](https://kustomize.io/) and [helm](https://helm.sh/).

Here we will use the following folder structure:

```shell
core
├── calico
├── metallb-system
└── metrics-server

```


#### Calico as CNI plugin via tigera-operator

Deploying Calico as the CNI (Container Network Interface) plugin will allows my Kubernetes cluster to manage networking and network policies effectively. I will use the `tigera-operator` Helm chart to install Calico.

```bash
mkdir -p core/calico
pushd core/calico/
cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization
namespace: tigera-operator

helmGlobals:
  chartHome: ../../helmCharts/

helmCharts:
- includeCRDs: true
  name: tigera-operator
  repo: https://docs.tigera.io/calico/charts
  version: 3.30.1
  releaseName: calico
  namespace: tigera-operator
  valuesFile: values.yaml
EOF


cat << EOF > values.yaml
installation:
  kubernetesProvider: ""  # No specific cloud provider
  cni:
    type: Calico
  calicoNetwork:
    bgp: Disabled                 # Use overlay instead of BGP
    natOutgoing: Enabled
    ipPools:
      - cidr: 10.0.0.0/16         # This must match with the Pod netwprk defined in part 1
        encapsulation: VXLAN      # Encapsulation for overlay networking
EOF
kustomize build . | kubectl apply -f -
popd
```

With this Calico configuration, we set up a VXLAN (Virtual Extensible LAN) overlay network for our Kubernetes cluster, which is suitable for a homelab environment without BGP (Broader Gateway Protocol) support. This encapsulation allows pods to communicate across different nodes by tunneling their traffic, without needing any changes to the underlying physical network.

#### MetalLB as load balancer

To expose my k8s applications to the local network, I deployed MetalLB. This provides the LoadBalancer service type that is typically only available in cloud environments such as AWS or GCP. The MetalLB setup will allow my nginx-ingress controller to receive a specified IP address from my local network so I can access my applications from other devices in my home network.

```bash
mkdir -p core/metallb
pushd core/metallb
cat << EOF > kustomozation.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization
namespace: metallb-system


helmGlobals:
  chartHome: ../../helmCharts/

resources:
- namespace.yaml
- ipaddress-pool.yaml

helmCharts:
- includeCRDs: true
  name: metallb
  repo: https://metallb.github.io/metallb
  version: 0.14.9
  releaseName: metrics-server
  namespace: metallb-system
  valuesFile: values.yaml
EOF

cat << EOF > values.yaml
controller:
  logLevel: debug
speaker:
  logLevel: debug
EOF

cat << EOF > namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: metallb-system
EOF

cat << EOF > ipaddress-pool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: homelab-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.0.210-192.168.0.221
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: homelab-advert
  namespace: metallb-system
spec:
  ipAddressPools:
  - homelab-pool
EOF
kustomize build . | kubectl apply -f -
popd
```

In my configuration, I defined an IPAddressPool (192.168.0.210-192.168.0.221), which is a range of IP addresses on my home's local network that I let MetalLB claim and assign to services.
The crucial part of my setup is the L2Advertisement, which I used to configure MetalLB for Layer 2 mode. In this mode, a speaker pod on one of my cluster nodes "claims" a service's IP address by responding to ARP requests on the local network. Instead of relying on my router to direct traffic, I use a combination of external-dns, Pi-hole and nginx-ingress as a LoadBalancer (see here). When MetalLB assigns an IP to a new service, external-dns automatically creates a DNS record in Pi-hole.
I chose this Layer 2 and DNS-based approach because it's the perfect counterpart to Calico's VXLAN mode. Both methods are ideal for my homelab since they're self-contained and don't require me to make any special configurations on my network router.

#### Metric-server for resource metrics

Finally, I will deploy the metrics-server to enable resource metrics collection in my cluster. This is essential for monitoring and autoscaling purposes.

```bash
mkdir -p core/metrics-server
pushd core/metrics-server

cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization


helmGlobals:
  chartHome: ../../helmCharts/

helmCharts:
- includeCRDs: true
  name: metrics-server
  repo: https://kubernetes-sigs.github.io/metrics-server/
  version: 3.12.2
  releaseName: metrics-server
  namespace: kube-system
  valuesFile: values.yaml
EOF

cat << EOF > values.yaml
apiService:
  insecureSkipTLSVerify: true
args:
  - --kubelet-insecure-tls
EOF
```

Quick check if the metrics-server is running:

```bash
kubectl get pods -n kube-system | grep metrics-server
kubectl top nodes
kubectl top pods -A
```

You should see an output similar to this:

```bash
~/git/NovoG93.github.io $ kubectl get pods -n kube-system | grep metrics-server
metrics-server-6f665c9d54-c7pt2   1/1     Running   0              25h

~/git/NovoG93.github.io $ kubectl top nodes
NAME   CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
cp1    207m         5%       3414Mi          58%
wp1    312m         5%       5861Mi          75%
wp2    83m          1%       3461Mi          59%
```

### Tools

Similar to the core components, I will use kustomize and helm to deploy the following tools:

```shell
tools
├── argocd
├── cert-manager
├── external-dns
├── external-secret-operator
├── homarr
├── jenkins
├── kubeshark
├── kyverno
├── nfs-provisioner
├── nginx-ingress
├── pihole
├── smb-provisioner
├── tailscale
├── vault
└── wildcard-tls
```


#### ArgoCD for GitOps

ArgoCD is a declarative, GitOps continuous delivery tool for Kubernetes. It will help me manage and automate the deployment of applications, tools and infrastructure components in my cluster.

```bash
