---
title: 'Setting up a Kubernetes Cluster the Hard Way with kubeadm (GitOps Series, Part 2)'
date: 2025-09-05
permalink: /posts/2025/09/05/homelab-part2/
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
      0. Enabling peristent storage via NFS and SMB provisioner
      1. Vault + External Secrets Operator for managing secrets
      2. ArgoCD for GitOps
      3. Enabling NFS for persistent storage
      4. Configuring cert-manager for managing TLS certificates
      5. Deploying nginx-ingress controller for ingress management
      6. Configuring kyverno to populate ingress resources with TLS certificates
      7. Installing pi-hole for DNS management and ad-blocking
      8. Configuring external-dns for dynamic DNS updates via pi-hole

During this setup we will run into the chicken-and-egg problem on a couple of occasions:
- We want to use ArgoCD to manage our k8s manifests in a GitOps fashion, but we need to deploy ArgoCD first. $\rightarrow$ This will be solved by deploying ArgoCD manually first, then using it to manage itself and the rest of the applications.
- Many of our applications rely on providing sensitive information as secrets, but we want to manage our secrets in Git as well. $\rightarrow$ This will be solved by deploying Vault and the External Secrets Operator.
- We want to use TLS certificates for secure communication, but we need cert-manager to manage our certificates. $\rightarrow$ This will be solved by deploying cert-manager first, then using it to issue a self-signed wildcard certificate for our domain.


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

You can find my repository [here](https://github.com/NovoG93/homelab) at [![Github Repo](https://img.shields.io/badge/--blue?logo=github)](https://github.com/NovoG93/homelab). If you want to follow along, feel free to fork it and adapt it to your needs. I will assume you have downloaded or forked the repository and cloned it to your local machine for the rest of this tutorial.

> I pin chart versions below. Feel free to bump later - I just kept them pinned for reproducibility.

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

Now we will setup core infrastructure components that are essential for the operation of the Kubernetes cluster. These components include a CNI (Container Network Interface) (calico) plugin for networking, a load balancer (metallb) for service exposure, and a metrics server (metrics-server) for resource monitoring. As they are fundamental to the cluster's operation, I will schedule them to run on the control-plane node.

Here we will use the following folder structure:

```shell
core
├── calico
├── metallb-system
└── metrics-server
```

#### Calico as CNI plugin via tigera-operator

Deploying Calico as the CNI (Container Network Interface) plugin will allows my Kubernetes cluster to manage networking and network policies effectively. I will use the `tigera-operator` Helm chart to install Calico.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [core/calico](https://github.com/NovoG93/homelab/tree/main/core/calico)
{: .notice--info}


<details markdown="1">
<summary>Show Calico installation commands</summary>

```bash
pushd core/calico/
cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization

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
      - cidr: 10.0.0.0/16         # This must match with the Pod network defined in part 1
        encapsulation: VXLAN      # Encapsulation for overlay networking
EOF
kustomize build . | kubectl apply -f -
popd
```
</details>

With this Calico configuration, we set up a VXLAN (Virtual Extensible LAN) overlay network for our Kubernetes cluster, which is suitable for a homelab environment without BGP (Broader Gateway Protocol) support. This encapsulation allows pods to communicate across different nodes by tunneling their traffic, without needing any changes to the underlying physical network.

#### MetalLB as load balancer

To expose my k8s applications to the local network, I deployed MetalLB. This provides the LoadBalancer service type that is typically only available in cloud environments such as AWS or GCP. The MetalLB setup will allow my nginx-ingress controller to receive a specified IP address from my local network so I can access my applications from other devices in my home network.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [core/metallb](https://github.com/NovoG93/homelab/tree/main/core/metallb)

<details markdown="1">
<summary>Show MetalLB installation commands</summary>

```bash
mkdir -p core/metallb
pushd core/metallb
cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization
namespace: metallb-system

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
  nodeSelector:
    node-role.kubernetes.io/control-plane: ""
  tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Exists"
    effect: "NoSchedule"
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
</details>

In my configuration, I defined an IPAddressPool (192.168.0.210-192.168.0.221), which is a range of IP addresses on my home's local network that I let MetalLB claim and assign to services.
The crucial part of my setup is the L2Advertisement, which I used to configure MetalLB for Layer 2 mode. In this mode, a speaker pod on one of my cluster nodes "claims" a service's IP address by responding to ARP requests on the local network. Instead of relying on my router to direct traffic, I use a combination of external-dns, Pi-hole and nginx-ingress as a LoadBalancer (see here). When MetalLB assigns an IP to a new service, external-dns automatically creates a DNS record in Pi-hole.
I chose this Layer 2 and DNS-based approach because it's the perfect counterpart to Calico's VXLAN mode. Both methods are ideal for my homelab since they're self-contained and don't require me to make any special configurations on my network router.

#### Metric-server for resource metrics

Finally, I will deploy the metrics-server to enable resource metrics collection in my cluster. This is essential for monitoring and autoscaling purposes.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [core/metrics-server](https://github.com/NovoG93/homelab/tree/main/core/metrics-server)

<details markdown="1">
<summary>Show metrics-server installation commands</summary>


```bash
mkdir -p core/metrics-server
pushd core/metrics-server

cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization

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
nodeSelector: 
  node-role.kubernetes.io/control-plane: ""
tolerations:
- key: "node-role.kubernetes.io/control-plane"
  operator: "Exists"
  effect: "NoSchedule"
apiService:
  insecureSkipTLSVerify: true
args:
  - --kubelet-insecure-tls
EOF

kustomize build . | kubectl apply -f -
popd
```
</details>



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

Similar to the core components above, the tools will be deployed on a designated node (`wp1`) to keep them separate from application workloads and the core components. This separation helps in managing resources and ensures that the tools have the necessary resources to operate effectively without interference from application workloads.

Here we will use the following folder structure:

```shell
tools
├── argocd
├── cert-manager
├── external-dns
├── external-secret-operator
├── kyverno
├── nfs-provisioner
├── nginx-ingress
├── pihole
├── smb-provisioner
├── vault
└── wildcard-tls
```

The values.yaml files for the tools will contain node affinity and tolerations to ensure they are scheduled on the `wp1` node. Since this is quite repetitive, I will not show it in every example below, but you can find it in my [repository](https://github.com/NovoG93/homelab/tree/main/tools). Below I will only highlight the unique configurations for each tool and the correct sequence of installation so that we can use GitOps principles as soon as possible.


0. NFS and SMB provisioners
1. Vault
2. External Secrets Operator
3. Cert-Manager
4. wildcard TLS certificate
5. Nginx Ingress Controller
6. ArgoCD
7. Kyverno
8. Pi-hole
9. External-DNS


> Note - If I were to place each and every k8s manifest for this setup in this post, it would be way too long and cumbersome to read. Therefore I will only highlight the important parts and provide links to my [GitHub repository](https://github.com/NovoG93/homelab) where necessary.


## Enabling peristent storage via NFS and SMB provisioner
Before we deploy any applications, we need to set up persistent storage for our cluster. In this setup, I will use NFS (Network File System) and SMB (Server Message Block) provisioners to provide persistent storage for my applications.

### NFS Provisioner

To enable NFS-based persistent storage in my Kubernetes cluster, I will deploy the `nfs-subdir-external-provisioner`. This provisioner allows dynamic provisioning of Persistent Volumes (PVs) using an existing NFS server. The installation is quite straightforward using Helm and Kustomize.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [core/metallb](https://github.com/NovoG93/homelab/tree/main/core/metallb)


<details markdown="1">
<summary>Show nfs-provisioner installation commands</summary>


Please ensure to set the `NFS_SERVER_IP` and `NFS_SERVER_PATH` environment variables to point to your NFS server's IP address and the export path on your NFS server before applying the configuration below.
{: .notice--warning}

```bash
mkdir -p tools/nfs-provisioner
pushd tools/nfs-provisioner
cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization

helmCharts:
- includeCRDs: true
  name: nfs-subdir-external-provisioner
  repo: https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
  version: 4.0.18
  releaseName: nfs-provisioner
  namespace: nfs-provisioner
  valuesFile: values.yaml
EOF

cat << EOF > values.yaml
nfs:
  server: ${NFS_SERVER_IP}  # Replace with the IP address of your NFS server
  path: ${NFS_SERVER_PATH}  # Replace with the export path on your NFS server
  mountOptions:
  volumeName: nfs-subdir-external-provisioner-root
  reclaimPolicy: Retain
  
storageClass:
  create: true
  defaultClass: true
  name: nfs-client
  allowVolumeExpansion: true
  reclaimPolicy: Retain
  archiveOnDelete: false
  onDelete: retain
  accessModes: ReadWriteOnce
  volumeBindingMode: Immediate
  
resources:
  limits:
   cpu: 100m
   memory: 128Mi
  requests:
   cpu: 100m
   memory: 128Mi

tolerations:
  - key: "tools"
    operator: "Exists"
    effect: "NoSchedule"

affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: "tools"
              operator: "Exists"
EOF
kustomize build . | kubectl apply -n nfs-provisioner -f -
popd
```
</details>


### SMB Provisioner

To enable SMB-based persistent storage in my Kubernetes cluster, I will deploy the `smb-csi-driver`. This driver allows dynamic provisioning of Persistent Volumes (PVs) using an existing SMB server. Similar to the NFS provisioner, the installation is straightforward. Note, that for each PV a separate share must be created on the SMB Server and a corresponding secret must be created in the namespace where the PV will be used.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [tools/smb-provisioner](https://www.github.com/NovoG93/homelab/tree/main/tools/smb-provisioner)

<details markdown="1">
<summary>Show smb-provisioner installation commands</summary>

```bash
mkdir -p tools/smb-provisioner
pushd tools/smb-provisioner
cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization

helmCharts:
- includeCRDs: true
  name: nfs-subdir-external-provisioner
  repo: https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
  version: 4.0.18
  releaseName: nfs-provisioner
  namespace: nfs-provisioner
  valuesFile: values.yaml
EOF

cat << EOF > values.yaml
driver:
  name: smb.csi.k8s.io
controller:
  name: csi-smb-controller
  replicas: 1
  dnsPolicy: ClusterFirstWithHostNet
linux:
  enabled: true
  dsName: csi-smb-node # daemonset name
  dnsPolicy: ClusterFirstWithHostNet
windows:
  enabled: false
EOF
kustomize build . | kubectl apply -n smb-provisioner -f -
popd
```

</details>


## Vault + External Secrets Operator for managing secrets

The combination of External Secrets Operator (ESO) and HashiCorp Vault provides a robust solution for managing secrets in a Kubernetes environment. The conjunction of these tools allows for secure storage, retrieval, and management of sensitive information such as API keys, passwords, and certificates outside of the Kubernetes cluster. Therefore, also allowing to manage secrets in a GitOps fashion without directly storing them in Git.


## External Secrets Operator

I deployed the ESO first, as it will also provide a ServiceAccount (SA) for Vault to authenticate against.
Here it is important to explicitly remember the name of the SA and the namespace it is created in, as we will need it later when configuring Vault.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [tools/external-secrets-operator](https://github.com/NovoG93/homelab/tree/main/tools/external-secrets-operator)

<details markdown="1">
<summary>Show external-secrets-operator installation commands</summary>

```bash
mkdir -p tools/external-secrets-operator
pushd tools/external-secrets-operator
export EXTERNAL_SECRETS_OPERATOR_NAMESPACE="external-secrets-operator"
export VAULT_NAMESPACE="vault"
cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization

resources:
- eso-vault-sa.yaml
- cluster-secret-store/dev.yaml

helmCharts:
- includeCRDs: true
  name: external-secrets
  repo: https://charts.external-secrets.io
  version: v0.19.2
  releaseName: external-secrets
  namespace: ${EXTERNAL_SECRETS_OPERATOR_NAMESPACE}
  valuesFile: values.yaml
EOF
cat << EOF > values.yaml
...
namespaceOverride: ${EXTERNAL_SECRETS_OPERATOR_NAMESPACE}
...
EOF
cat << EOF > eso-vault-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eso-vault
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: eso-tokenrequest
rules:
  - apiGroups: [""]
    resources: ["serviceaccounts/token"]
    verbs: ["create"]
    resourceNames: ["eso-vault"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: eso-tokenrequest-binding
subjects:
  - kind: ServiceAccount
    name: external-secret-operator
    namespace: ${EXTERNAL_SECRETS_OPERATOR_NAMESPACE}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: eso-tokenrequest
EOF
cat << EOF > cluster-secret-store/dev.yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: vault-dev
spec:
  provider:
    vault:
      server: "http://vault.${VAULT_NAMESPACE}.svc:8200"
      path: "dev"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "eso"
          serviceAccountRef:
            name: "eso-vault"
            namespace: ${EXTERNAL_SECRETS_OPERATOR_NAMESPACE}
EOF
kustomize build . | kubectl apply -f -
popd
```
</details>

With this configuration, the External Secrets Operator is set up to interact with Vault using Kubernetes authentication. The `ClusterSecretStore` resource defines how the operator connects to Vault, specifying the server address, authentication method, and the role it will use to access secrets. Until Vault is configured, the ClusterSecretsStore will not be functional, but we can already deploy it to the cluster.


### Vault

Setting up Vault to manage secrets is a non-trivial task. It includes deploying Vault itself, initializing it, unsealing it, configuring policies, and setting up authentication methods. Below are the steps I took to deploy Vault using Helm and configure it for use with External Secrets Operator.

Please have a look at the init.sh script in my [repository](https://github.com/NovoG93/homelab/blob/6626bff6fde5182d5c3b9f10441cb8d6a4683c08/tools/vault/operator-init.sh) for the complete setup. The script automates the initialization, unsealing, and configuration of a HashiCorp Vault instance, including secret engines, policies, Kubernetes authentication, user management, and key backup. It is designed to be idempotent, safe to run multiple times, and adapts to whether it's executed inside a pod or via kubectl. Key environment variables control user creation and integration with External Secrets Operator, making it a complete Vault bootstrap for development use.
{: .notice--info}


[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [tools/vault](https://github.com/NovoG93/homelab/tree/main/tools/vault)

<details markdown="1">
<summary>Show vault installation commands</summary>

In my setup I am using a `sh` init script in combination with policies to initialize and unseal Vault. The script will be executed as a sidecar container in the Vault pod. This is not the most secure way to handle the unsealing process, but it is sufficient for a homelab setup. It will configure a root token (or re-use an existing one) and store it in a Persistent Volume Claim (PVC), generate a admin and read only policy, enable the kubernetes auth method and create a user authentication for accessing the Vault UI without the root token.


```bash
mkdir -p tools/vault
pushd tools/vault
export EXTERNAL_SECRETS_OPERATOR_NAMESPACE="external-secrets-operator"
export VAULT_NAMESPACE="vault"
export VAULT_ADMIN_USER="Georg"
export VAULT_ADMIN_PASSWORD="changeme"  # Change this to a secure password
cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization

resources:
- virtualserver.yaml
- vault-keys-pvc.yaml

configMapGenerator:
- name: vault-init-script
  files:
  - init.sh=operator-init.sh
- name: policies
  files:
  - policies/admin.hcl
  - policies/read-only.hcl

helmCharts:
- includeCRDs: true
  name: vault
  repo: https://helm.releases.hashicorp.com
  version: 0.30.0
  releaseName: vault
  namespace: vault
  valuesFile: values.yaml
EOF
cat << EOF > values.yaml
...

server:
  serviceAccount:
    create: true
    name: vault

  extraEnvironmentVars:
    VAULT_SKIP_VERIFY: "true"
    USERS: "${VAULT_ADMIN_USER}"
    USER_PASSWORD: "${VAULT_ADMIN_PASSWORD}"
    FORMAT: "" # -format=json

  postStart:
  - /bin/sh
  - -c
  - |
    chmod +x /tmp/init.sh || true
    nohup /tmp/init.sh > /tmp/keys/logs.txt 2>&1 &


  volumeMounts:
  - name: init-script
    mountPath: /tmp/init.sh
    subPath: init.sh
  - name: keys-json
    mountPath: /tmp/keys/
  - name: policies
    mountPath: /tmp/policies/


  volumes:
  - name: init-script
    configMap:
      name: vault-init-script
      defaultMode: 0755
  - name: policies
    configMap:
      name: policies
      defaultMode: 0555
  - name: keys-json
    persistentVolumeClaim:
      claimName: vault-keys-pvc
...

# Vault UI
ui:
  enabled: true
EOF
```

