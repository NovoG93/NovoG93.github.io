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

**← [Part 1: Setting up the Kubernetes cluster](/posts/2025/08/20/homelab-part1/)**

In the previous post of this series, we set up a Kubernetes cluster from scratch using kubeadm. Now that we have a functional cluster, it's time to deploy some essential infrastructure applications and tools to manage our cluster effectively.

Hence this post will now describe how to:

1. Setting up core infrastructure components and tools
   0. Adding taints and labels to the nodes
   1. Core
      1. Calico as CNI plugin via tigera-operator
      2. MetalLB as load balancer
      3. Metric-server for resource metrics
   2. Tools
      0. Enabling persistent storage via NFS and SMB provisioners
      1. External Secrets Operator for Kubernetes secret management
      2. Vault for secure secrets storage
      3. ArgoCD for GitOps continuous delivery
      4. Cert-manager for automated TLS certificate management
      5. Wildcard TLS certificate provisioning
      6. Nginx Ingress Controller with VirtualServer CRDs
      7. Kyverno for policy-based secret distribution
      8. Pi-hole for DNS management and ad-blocking
      9. External-DNS for automatic DNS record management

During this setup we will run into the chicken-and-egg problem on a couple of occasions:
- My applications rely on  secrets, but I want to manage my secrets in Git as well. $\rightarrow$ This will be solved by deploying Vault and the External Secrets Operator.
- I want to use ArgoCD to manage my k8s manifests in a GitOps fashion, but I need to deploy ArgoCD first. $\rightarrow$ This will be solved by deploying ArgoCD manually first, then using it to manage itself and the rest of the applications.


<div style="display: flex; justify-content: center; align-items: center; flex-wrap: wrap;">
    <figure style="margin: 10px;">
        <img src="/images/homelab/Cluster-Overview.drawio.png" alt="Cluster Overview" width="300">
        <figcaption>Cluster Overview: 1CP + 2WP nodes</figcaption>
    </figure>
</div>

The picture above illustrates the overall architecture of my homelab cluster. It consists of one control-plane (CP) node and two worker (WP) nodes. The core infrastructure components are deployed on the control-plane node, while the tools and applications are distributed across the worker nodes based on their designated types and the corresponding taints and labels on the CP.


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

Now we will setup core infrastructure components that are essential for the operation of the Kubernetes cluster. These components include a CNI (Container Network Interface) (`calico`) plugin for networking, a load balancer (`metallb`) for service exposure, and a metrics server (`metrics-server`) for resource monitoring. As they are fundamental to the cluster's operation, they will be deployed as DaemonSets and therefore run on all nodes, including the control-plane node.

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
export POD_CIDR="10.0.0.0/16"  # This must match with the Pod network defined in part 1
mkdir -p core/calico
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
      - cidr: $POD_CIDR           # This must match with the Pod network defined in part 1
        encapsulation: VXLAN      # Encapsulation for overlay networking
EOF
kustomize build . | kubectl apply -f -
popd
```
</details>

With this Calico configuration, we set up a VXLAN (Virtual Extensible LAN) overlay network for our Kubernetes cluster, which is suitable for a homelab environment without BGP (Broader Gateway Protocol) support. This encapsulation allows pods to communicate across different nodes by tunneling their traffic, without needing any changes to the underlying physical network.

#### MetalLB as load balancer

To expose my k8s applications to the local network, I deployed MetalLB. It will provide the the LoadBalancer service type. This MetalLB setup will allow my nginx-ingress controller to receive a specified IP address from my local network so I can access my applications from other devices in my home network.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [core/metallb-system](https://github.com/NovoG93/homelab/tree/main/core/metallb-system)

<details markdown="1">
<summary>Show MetalLB installation commands</summary>

```bash
export METALLB_IP_RANGE="192.168.0.210-192.168.0.221"  # Change this to a range suitable for your local network
mkdir -p core/metallb-system
pushd core/metallb-system
cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization

resources:
- namespace.yaml
- ipaddress-pool.yaml

helmCharts:
- includeCRDs: true
  name: metallb
  repo: https://metallb.github.io/metallb
  version: 0.14.9
  releaseName: metallb
  namespace: metallb-system
  valuesFile: values.yaml
EOF

cat << EOF > values.yaml
controller:
  logLevel: debug
  tolerations:
    - operator: "Exists" # Tolerate any taint to run on all nodes
speaker:
  logLevel: debug
  tolerations:
    - operator: "Exists" # Tolerate any taint to run on all nodes
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
  - $METALLB_IP_RANGE
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
The crucial part here is the L2Advertisement, which I used to configure MetalLB for a Layer 2 mode. In this mode, a speaker pod on one of my cluster nodes "claims" a service's IP address by responding to ARP requests on the local network. As I describe below, instead of relying on my router to direct traffic, I use a combination of external-dns, Pi-hole and nginx-ingress as a LoadBalancer. When a new k8s.nginx.io/v1 VirtualServer is created the external-dns automatically creates a DNS record in Pi-hole.
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

The tools will be deployed on a designated node (`wp1`) to keep them separate from application workloads.

Here I am using the following folder structure:

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

To schedule the tools on the `wp1` node, I will use node affinity and tolerations to ensure they are scheduled on the `wp1` node. The sequence in which the tools are deployed is a crucial aspect of this setup, as some tools depend on others to be functional first. Therefore, I will deploy the tools in the following order:

| # | Tool | Purpose | Mandatory Dependencies | Optional Dependencies |
|---|------|---------|------------------------|----------------------|
| 0 | NFS and SMB provisioners | Provides persistent storage for applications | None | None |
| 1 | External Secrets Operator | Integrates external secret management systems | None | Vault |
| 2 | Vault | Manages secrets and sensitive data | NFS/SMB provisioners | Nginx Ingress Controller (UI access without port-forwarding) |
| 3 | ArgoCD | Continuous delivery tool for Kubernetes | External Secrets Operator, NFS/SMB provisioners | Nginx Ingress Controller (UI access without port-forwarding) |
| 4 | Cert-Manager | Automates the management and issuance of TLS certificates | External Secrets Operator | None |
| 5 | Wildcard TLS certificate | Contains a helm chart to create a wildcard TLS certificate | Cert-Manager | None |
| 6 | Nginx Ingress Controller | Manages external access to services | Wildcard TLS certificate | None |
| 7 | Kyverno | Policy engine for Kubernetes | None | Cert-Manager, Wildcard TLS certificate |
| 8 | External-DNS | Manages DNS records for Kubernetes resources | Nginx Ingress Controller | Pi-hole |
| 9 | Pi-hole | Network-wide ad blocker and DNS server | External-DNS | Nginx Ingress Controller (UI access without port-forwarding) |


> Note - If I were to place each and every k8s manifest for this setup in this post, it would be way too long and cumbersome to read. Therefore I will only highlight the important parts and provide links to my [GitHub repository](https://github.com/NovoG93/homelab) where necessary.



#### Enabling persistent storage via NFS and SMB provisioner
Before we deploy any applications, we need to set up persistent storage for our cluster. In this setup, I will use NFS (Network File System) and SMB (Server Message Block) provisioners to provide persistent storage for my applications.

##### NFS Provisioner

To enable NFS-based persistent storage in my Kubernetes cluster, I will deploy the `nfs-subdir-external-provisioner`. This provisioner allows dynamic provisioning of Persistent Volumes (PVs) using an existing NFS server. The installation is quite straightforward using Helm and Kustomize.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [tools/nfs-provisioner](https://github.com/NovoG93/homelab/tree/main/tools/nfs-provisioner)


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
EOF
kustomize build . | kubectl apply -n nfs-provisioner -f -
popd
```
</details>


##### SMB Provisioner

To enable SMB-based persistent storage in my Kubernetes cluster, I will deploy the `csi-driver-smb`. This CSI driver allows dynamic provisioning of Persistent Volumes (PVs) using an existing SMB server. Similar to the NFS provisioner, the installation is straightforward. Note, that for each PV a separate share must be created on the SMB Server and a corresponding secret must be created in the namespace where the PV will be used.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [tools/smb-provisioner](https://github.com/NovoG93/homelab/tree/main/tools/smb-provisioner)

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
  name: csi-driver-smb
  repo: https://raw.githubusercontent.com/kubernetes-csi/csi-driver-smb/master/charts
  version: v1.18.0
  releaseName: csi-driver-smb
  namespace: smb-provisioner
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


### External Secrets Operator + Vault for managing secrets

The combination of External Secrets Operator (ESO) and HashiCorp Vault provides a robust solution for managing secrets in a Kubernetes environment. The conjunction of these tools allows for secure storage, retrieval, and management of sensitive information such as API keys, passwords, and certificates outside of the Kubernetes cluster. Therefore, also allowing to manage secrets in a GitOps fashion without directly storing them in Git.


#### External Secrets Operator

I deployed the ESO first, as it will also provide a ServiceAccount (SA) for Vault to authenticate against.
Here it is important to explicitly remember the name of the SA and the namespace it is created in, as they are needed later during configuring Vault.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [tools/external-secret-operator](https://github.com/NovoG93/homelab/tree/main/tools/external-secret-operator)

<details markdown="1">
<summary>Show external-secrets-operator installation commands</summary>

```bash
mkdir -p tools/external-secret-operator
pushd tools/external-secret-operator
export EXTERNAL_SECRETS_OPERATOR_NAMESPACE="external-secret-operator"
export VAULT_NAMESPACE="vault"
cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization

resources:
- eso-vault-sa.yaml
- cluster-secret-store/dev.yaml
- cluster-secret-store/prod.yaml

helmCharts:
- includeCRDs: true
  name: external-secrets
  repo: https://charts.external-secrets.io
  version: v1.0.0
  releaseName: external-secrets
  namespace: ${EXTERNAL_SECRETS_OPERATOR_NAMESPACE}
  valuesFile: values.yaml
EOF

cat << EOF > values.yaml
global:
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

namespaceOverride: ${EXTERNAL_SECRETS_OPERATOR_NAMESPACE}
revisionHistoryLimit: 5
installCRDs: true
EOF

cat << EOF > eso-vault-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eso-vault
---
# Let the ESO controller (serviceaccount: external-secret-operator) create TokenRequests for the SA above
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
    name: external-secret-operator   # ESO controller's SA (default from Helm chart)
    namespace: ${EXTERNAL_SECRETS_OPERATOR_NAMESPACE}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: eso-tokenrequest
EOF

mkdir -p cluster-secret-store
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
kustomize build . | kubectl apply -n vault -f -
popd
```
</details>

With this configuration, the External Secrets Operator is set up to interact with Vault using Kubernetes authentication. The `ClusterSecretStore` resource defines how the operator connects to Vault, specifying the server address, authentication method, and the role it will use to access secrets. Until Vault is configured, the ClusterSecretsStore will not be functional, but we can already deploy it to the cluster.


#### Vault

Setting up Vault to manage secrets is a non-trivial task. It includes deploying Vault itself, initializing it, unsealing it, configuring policies, and setting up authentication methods. Below are the steps I took to deploy Vault using Helm and configure it for use with External Secrets Operator.

Please have a look at the init.sh script in my [repository](https://github.com/NovoG93/homelab/blob/main/tools/vault/operator-init.sh) for the complete setup. The script automates the initialization, unsealing, and configuration of a HashiCorp Vault instance, including secret engines, policies, Kubernetes authentication, user management, and key backup. It is designed to be idempotent, safe to run multiple times, and adapts to whether it's executed inside a pod or via kubectl. Key environment variables control user creation and integration with External Secrets Operator, making it a complete Vault bootstrap for development use.
{: .notice--info}


[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [tools/vault](https://github.com/NovoG93/homelab/tree/main/tools/vault)

<details markdown="1">
<summary>Show vault installation commands</summary>

In my setup I am using a init script in combination with policies to initialize and unseal Vault. The script will be executed as a sidecar container in the Vault pod. This is not the most secure way to handle the unsealing process, but it is sufficient for a homelab setup. It will configure a root token (or re-use an existing one) and store it in a Persistent Volume Claim (PVC), generate a admin and read only policy, enable the kubernetes auth method and create a user authentication for accessing the Vault UI without the root token.


```bash
mkdir -p tools/vault
pushd tools/vault
export EXTERNAL_SECRETS_OPERATOR_NAMESPACE="external-secrets-operator"
export VAULT_NAMESPACE="vault"
export VAULT_ADMIN_USER="Admin"
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
</details>


We can now create a test secret in Vault to verify that everything is working as expected:

```bash
ROOT_TOKEN="$(kubectl exec -n vault pod/vault-0 -- cat /tmp/keys/keys.json | grep -i "root token" | awk -F \\: '{print $2}')"
kubectl exec -n vault vault-0 -- /bin/sh -c "vault login ${ROOT_TOKEN} && vault kv put dev/test/test-secret test-secret=my-secret-value"
```

You should see an output similar to this:

```bash
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                hvs.XXXXXXXXXXXX
token_accessor       XXXXXXXXXXXXXXXX
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
====== Secret Path ======
dev/data/test/test-secret

======= Metadata =======
Key                Value
---                -----
created_time       2025-11-05T15:45:48.1366156Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

=============
```

When looking at the vault ui (use `kubectl  port-forward -n vault vault-0 8200:8200` to access it) you should now be able to login with the user `Admin` and the password you set above, or use the root token login method. Further, you should see the secret we just created under `dev/test/test-secret`.

Lastly, to test the External-Secret-Operator to Vault connection I used the commands below to create a ExternalSecret via a heredoc:

```bash
➜  ~ cat << EOF | kubectl apply -n default -f -
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: test-secret
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: vault-dev
    kind: ClusterSecretStore
  target:
    name: test-secret
    deletionPolicy: Delete
  data:
    - secretKey: test-secret
      remoteRef:
        conversionStrategy: Default
        decodingStrategy: None
        metadataPolicy: None
        key: dev/test/test-secret
        property: test-secret
EOF
externalsecret.external-secrets.io/test-secret created
➜  ~ kubectl get secret -n default test-secret -ojsonpath='{.data.test-secret}' | base64 -d
my-secret-value
```


### ArgoCD for GitOps


ArgoCD is a declarative, GitOps continuous delivery tool for Kubernetes. It allows managing Kubernetes resources using Git repositories as the source of truth. There are many options to implement GitOps patterns using ArgoCD, I chose to use a combination of [ApplicationSets](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/) with a [GitGenerator](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Git/) and a single Apps-of-Apps pattern to manage my applications.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [tools/argocd](https://github.com/NovoG93/homelab/tree/main/tools/argocd)

#### Application Definition with app.yaml

Each application or tool in my setup includes an `app.yaml` file that defines its metadata. This file is used by ArgoCD's ApplicationSet GitGenerator to automatically discover and create ArgoCD Applications. The structure is simple:

```yaml
# Example: apps/immich/app.yaml
name: immich
path: apps/immich
namespace: immich
project: default
```

The ApplicationSet scans the repository for these `app.yaml` files and uses the values to create corresponding ArgoCD Applications. This pattern allows for easy addition of new applications - simply create a folder with the necessary Kubernetes manifests and an `app.yaml` file, and ArgoCD will automatically pick it up.

<details markdown="1">
<summary>Show argocd installation commands</summary>

```bash
mkdir -p tools/argocd
pushd tools/argocd
cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization
namespace: argocd

resources:
- virtualserver.yaml
- appSets

helmCharts:
- includeCRDs: true
  name: argo-cd
  repo: https://argoproj.github.io/argo-helm
  version: 8.0.14
  releaseName: argocd
  namespace: argocd
  valuesFile: values.yaml

patches:
- path: patches/kustomize.yaml
EOF

cat << EOF > values.yaml
global:
  domain: argocd.novotny.live
  tolerations:
    - key: "tools"
      operator: "Exists"
      effect: "NoSchedule"
  affinity:
    nodeAffinity:
      type: hard
      matchExpressions:
      - key: "tools"
        operator: "Exists"

configs:
  params:
    server.insecure: true

server:
  ingress:
    enabled: false

controller:
  resources:
    limits:
      cpu: 4000m
    requests:
      cpu: 250m
      memory: 256Mi
EOF

# Create the kustomize patch for ArgoCD ConfigMap
mkdir -p patches
cat << EOF > patches/kustomize.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  kustomize.buildOptions: "--enable-helm --load-restrictor=LoadRestrictionsNone"
  applicationsetcontroller.enable.new.git.file.globbing: "true"
EOF

# Create ApplicationSets for each application category
mkdir -p appSets
cat << EOF > appSets/tools.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: tools-appset
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - git:
        repoURL: https://github.com/NovoG93/homelab
        revision: main
        files:
          - path: "tools/**/app.yaml"
  template:
    metadata:
      name: "{% raw %}{{ .name }}{% endraw %}"
      labels:
        group: tools
    spec:
      project: "{% raw %}{{ .project }}{% endraw %}"
      source:
        repoURL: https://github.com/NovoG93/homelab
        targetRevision: main
        path: "{% raw %}{{ .path.path }}{% endraw %}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "{% raw %}{{ .namespace }}{% endraw %}"
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
          - PrunePropagationPolicy=foreground
          - ApplyOutOfSyncOnly=true
          - ServerSideApply=true
EOF

kustomize build . | kubectl apply -n argocd -f -
popd
```
</details>


My configuration of ArgoCD is pretty basic. Below are a few notable configurations specific to my setup defined in the argocd-cm ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  ...
data:
  kustomize.buildOptions: "--enable-helm --load-restrictor=LoadRestrictionsNone"
  applicationsetcontroller.enable.new.git.file.globbing: "true"
```

* `kustomize.buildOptions`: this provides build flags to Argos `kustomize` to allow working with helm charts propperly
* `applicationsetcontroller.enable.new.git.file.globbing`: this enables the new globbing implementation for the GitGenerator go template methods.


To deploy I use a combination of an ApplicationSet with a GitGenerator and an Apps-of-Apps pattern. The ApplicationSet is defined in the `appSets` folder and points to my git repository as the source of truth for all applications. The GitGenerator scans the `apps` folder in my repository for application definitions and creates an ArgoCD Application for each one.


### Cert-Manager for TLS Certificate Management

Cert-Manager automates the management and issuance of TLS certificates in Kubernetes. I configured it to use Let's Encrypt with DNS01 challenge via Cloudflare for obtaining wildcard certificates.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [tools/cert-manager](https://github.com/NovoG93/homelab/tree/main/tools/cert-manager)

<details markdown="1">
<summary>Show cert-manager installation commands</summary>

```bash
mkdir -p tools/cert-manager
pushd tools/cert-manager
cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization

resources:
- secret-api.yaml
- issuer.yaml

helmCharts:
- includeCRDs: true
  name: cert-manager
  repo: https://charts.jetstack.io
  version: v1.17.2
  releaseName: cert-manager
  namespace: cert-manager
  valuesFile: values.yaml
EOF

cat << EOF > values.yaml
crds:
  enabled: true
global:
  leaderElection:
    namespace: cert-manager
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
# Same tolerations and affinity for webhook, cainjector, and startupapicheck
webhook:
  affinity: ...
  tolerations: ...
cainjector:
  affinity: ...
  tolerations: ...
EOF

# Create ClusterIssuer for Let's Encrypt with Cloudflare DNS challenge
cat << EOF > issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns-cloudflare
spec:
  acme:
    email: your-email@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
    - dns01:
        cloudflare:
          apiTokenSecretRef:
            name: api-token
            key: api-token
EOF
kustomize build . | kubectl apply -n cert-manager -f -
popd
```
</details>


### Wildcard TLS Certificate

To avoid requesting a new certificate for each service, I use a wildcard certificate that covers all subdomains of my domain. This certificate is generated once by cert-manager and distributed to other namespaces via Kyverno policies.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [tools/wildcard-tls](https://github.com/NovoG93/homelab/tree/main/tools/wildcard-tls)

<details markdown="1">
<summary>Show wildcard-tls installation commands</summary>

```bash
mkdir -p tools/wildcard-tls/certificate-provision/templates
pushd tools/wildcard-tls
cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization

helmCharts:
- name: certificate-provision
  version: 0.1.0
  releaseName: certificate-provision
  namespace: wildcard-tls
  valuesFile: values.yaml
EOF

cat << EOF > values.yaml
certificate:
  commonName: "novotny.live"
  dnsNames:
  - "*.novotny.live"
  - "novotny.live"
  issuerKind: "ClusterIssuer"
  issuerName: "letsencrypt-dns-cloudflare"
  secretName: wildcard-tls
EOF
kustomize build . | kubectl apply -n wildcard-tls -f -
popd
```
</details>


### Nginx Ingress Controller

I use the NGINX Ingress Controller with VirtualServer CRDs for managing external access to services. It integrates with MetalLB to receive a static LoadBalancer IP and with external-dns for automatic DNS record management.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [tools/nginx-ingress](https://github.com/NovoG93/homelab/tree/main/tools/nginx-ingress)

<details markdown="1">
<summary>Show nginx-ingress installation commands</summary>

```bash
mkdir -p tools/nginx-ingress
pushd tools/nginx-ingress
cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization

helmCharts:
- includeCRDs: true
  name: nginx-ingress
  repo: https://helm.nginx.com/stable
  version: 1.1.2
  releaseName: nginx-ingress
  namespace: nginx-ingress
  valuesFile: values.yaml
EOF

cat << EOF > values.yaml
controller:
  ingressClass:
    name: "nginx"
    setAsDefaultIngress: true
  enableExternalDNS: true
  healthStatus: true
  enableCustomResources: true
  enableSnippets: true
  enableTLSPassthrough: true
  wildcardTLS:
    secret: nginx-ingress/wildcard-tls
  service:
    type: LoadBalancer
    loadBalancerIP: "192.168.0.210"  # Static IP from MetalLB pool
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
kustomize build . | kubectl apply -n nginx-ingress -f -
popd
```
</details>

#### VirtualServer Example

Each application exposes itself via a VirtualServer CRD that defines the routing and TLS configuration:

```yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: argocd
  namespace: argocd
  annotations:
    external-dns.alpha.kubernetes.io/hostname: "argocd.novotny.live"
spec:
  externalDNS:
    enable: true
  host: argocd.novotny.live
  tls:
    secret: wildcard-tls
  upstreams:
  - name: argocd-server
    service: argocd-server
    port: 80
  routes:
  - path: /
    action:
      pass: argocd-server
```


### Kyverno for Policy-Based Secret Distribution

Kyverno is a policy engine for Kubernetes that I use to automatically copy the wildcard TLS certificate to all namespaces that need it. I opted for this method to allow for rapid distribution of the certificate without manual intervention or getting blocked by the request limits of Let's Encrypt.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [tools/kyverno](https://github.com/NovoG93/homelab/tree/main/tools/kyverno)

<details markdown="1">
<summary>Show kyverno installation commands</summary>

```bash
mkdir -p tools/kyverno/clusterPolicies
pushd tools/kyverno
cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization

resources:
- clusterPolicies

helmCharts:
- includeCRDs: true
  name: kyverno
  repo: https://kyverno.github.io/kyverno
  version: 3.5.2
  releaseName: kyverno
  namespace: kyverno
  valuesFile: values.yaml
EOF

cat << EOF > values.yaml
global:
  tolerations:
    - key: "tools"
      operator: "Exists"
      effect: "NoSchedule"
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: "tools"
              operator: "Exists"
EOF

# Create ClusterPolicy to copy wildcard certificate to all namespaces
cat << EOF > clusterPolicies/copy-wildcard-secret.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: copy-wildcard-secret
spec:
  validationFailureAction: Enforce
  useServerSideApply: true
  rules:
  - name: copy-wildcard-secret
    skipBackgroundRequests: false
    match:
      any:
      - resources:
          kinds:
          - Namespace
    exclude:
      any:
      - resources:
          kinds: ["Namespace"]
          names:
          - "!wildcard-tls"
          - "!kube-system"
          - "!kube-public"
          # ... other system namespaces
    generate:
      apiVersion: v1
      kind: Secret
      name: wildcard-tls
      namespace: "{% raw %}{{request.object.metadata.name}}{% endraw %}"
      synchronize: true
      generateExisting: true
      clone:
        name: wildcard-tls
        namespace: wildcard-tls
EOF
kustomize build . | kubectl apply -n kyverno -f -
popd
```
</details>


### External-DNS for Automatic DNS Management

External-DNS automatically manages DNS records based on Kubernetes resources. I configured it to work with Pi-hole as the DNS provider, creating A records for each VirtualServer.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [tools/external-dns](https://github.com/NovoG93/homelab/tree/main/tools/external-dns)

<details markdown="1">
<summary>Show external-dns installation commands</summary>

```bash
mkdir -p tools/external-dns
pushd tools/external-dns
cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization

resources:
- es-pihole-password.yaml

helmCharts:
- includeCRDs: true
  name: external-dns
  repo: https://kubernetes-sigs.github.io/external-dns/
  version: 1.19.0
  releaseName: external-dns
  namespace: external-dns
  valuesFile: values.yaml
EOF

cat << EOF > values.yaml
provider: pihole

env:
  - name: EXTERNAL_DNS_PIHOLE_SERVER
    value: "http://pihole-web.pihole.svc.cluster.local"
  - name: EXTERNAL_DNS_PIHOLE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: pihole-password
        key: password

policy: upsert-only
registry: noop
logLevel: debug

domainFilters:
  - novotny.live

sources:
  - crd
  - service
  - ingress

extraArgs:
  - --crd-source-apiversion=externaldns.nginx.org/v1
  - --crd-source-kind=DNSEndpoint

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
kustomize build . | kubectl apply -n external-dns -f -
popd
```
</details>


### Pi-hole for DNS Management and Ad-Blocking

Pi-hole serves as my local DNS server, handling internal DNS resolution and ad-blocking. External-DNS creates records in Pi-hole for all my Kubernetes services.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [tools/pihole](https://github.com/NovoG93/homelab/tree/main/tools/pihole)

<details markdown="1">
<summary>Show pihole installation commands</summary>

```bash
mkdir -p tools/pihole
pushd tools/pihole
cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization

resources:
- virtualserver.yaml
- es-pihole-password.yaml

helmCharts:
- includeCRDs: true
  name: pihole
  repo: https://mojo2600.github.io/pihole-kubernetes/
  version: 2.31.0
  releaseName: pihole
  namespace: pihole
  valuesFile: values.yaml
EOF

cat << EOF > values.yaml
serviceDhcp:
  enabled: false

serviceDns:
  type: LoadBalancer
  loadBalancerIP: "192.168.0.211"  # Static IP from MetalLB pool
  externalTrafficPolicy: Cluster
  annotations:
    metallb.universe.tf/allow-shared-ip: pihole-svc

serviceWeb:
  http:
    enabled: true
  https:
    enabled: true
  type: ClusterIP

admin:
  enabled: true
  existingSecret: "pihole-password"
  passwordKey: "password"

DNS1: "8.8.8.8"
DNS2: "8.8.4.4"

persistentVolumeClaim:
  enabled: true
  size: "500Mi"

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
kustomize build . | kubectl apply -n pihole -f -
popd
```
</details>

#### Verifying the DNS Flow

Once all components are deployed, you can verify the complete flow works:

```bash
# Check that external-dns has created records in Pi-hole
kubectl exec -n pihole pihole-xxx -- cat /etc/pihole/hosts/custom.list | grep 192.168
# Expected output:
# 192.168.0.210 argocd.novotny.live
# 192.168.0.210 pihole.novotny.live
# 192.168.0.210 vault.novotny.live
# 192.168.0.210 immich.novotny.live

# Test DNS resolution
kubectl exec -n default debug -- dig @192.168.0.211 argocd.novotny.live
# Should return: 192.168.0.210 (nginx-ingress LoadBalancer IP)

# Verify service IPs
kubectl get svc -n nginx-ingress
# nginx-ingress-controller  LoadBalancer  ...  192.168.0.210  80:xxx/TCP,443:xxx/TCP

kubectl get svc -n pihole
# pihole-dns-tcp  LoadBalancer  ...  192.168.0.211  53:xxx/TCP
# pihole-dns-udp  LoadBalancer  ...  192.168.0.211  53:xxx/UDP
```


## Summary

In this post, we covered the deployment of essential infrastructure components and tools for a Kubernetes homelab:

| Component | Purpose | Key Features |
|-----------|---------|--------------|
| **Calico** | CNI Plugin | VXLAN overlay networking, network policies |
| **MetalLB** | Load Balancer | L2 mode, static IP allocation for services |
| **Metrics Server** | Resource Monitoring | Enables `kubectl top` and autoscaling |
| **NFS/SMB Provisioners** | Persistent Storage | Dynamic PV provisioning from NAS |
| **External Secrets Operator** | Secret Management | Kubernetes-native secret sync with Vault |
| **Vault** | Secrets Backend | Secure storage, user management, policies |
| **ArgoCD** | GitOps CD | ApplicationSets, Git-based deployments |
| **Cert-Manager** | TLS Management | Let's Encrypt integration, auto-renewal |
| **Nginx Ingress** | Traffic Routing | VirtualServer CRDs, TLS termination |
| **Kyverno** | Policy Engine | Auto-distribute TLS secrets |
| **External-DNS** | DNS Management | Auto-create DNS records in Pi-hole |
| **Pi-hole** | DNS Server | Local DNS, ad-blocking |

The key architectural decisions were:
1. **Separation of concerns**: Core components on all nodes, tools on dedicated `wp1` node, apps on `wp2`
2. **GitOps-first**: All configurations stored in Git, managed by ArgoCD
3. **Secret management**: Vault as the source of truth, ESO syncs to Kubernetes
4. **Self-contained networking**: MetalLB L2 + Pi-hole DNS means no router configuration needed

In the next part, we'll cover deploying actual applications using the App of Apps pattern and setting up secure remote access with Tailscale.

---

**← [Part 1: Setting up the Kubernetes cluster](/posts/2025/08/20/homelab-part1/)** | **[Part 3: App of Apps and Applications →](/posts/2025/11/30/homelab-part3/)**