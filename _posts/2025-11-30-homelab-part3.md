---
title: 'Setting up a Kubernetes Cluster the Hard Way with kubeadm (GitOps Series, Part 3)'
date: 2025-11-30
permalink: /posts/2025/11/30/homelab-part3/
tags:
    - kubernetes
    - gitops
    - kubeadm
    - homelab
    - devops
    - cloud-native
---

# Setting up a Kubernetes Cluster the Hard Way with kubeadm (GitOps Series, Part 3)

**← [Part 1: Setting up the Kubernetes cluster](/posts/2025/08/20/homelab-part1/)** | **← [Part 2: Core Infrastructure and Tools](/posts/2025/09/05/homelab-part2/)**

In the previous parts of this series, we set up a Kubernetes cluster from scratch using kubeadm (Part 1) and deployed essential infrastructure components and tools (Part 2). Now it's time to put everything together and deploy actual applications using the App of Apps pattern with ArgoCD.

This post will cover:

1. Understanding the App of Apps pattern with ArgoCD ApplicationSets
2. Deploying CloudNative-PG for PostgreSQL databases
3. Deploying Tailscale Operator for secure remote access
4. Adding Kyverno policies for automatic Tailscale ingress generation
5. Deploying Immich as a self-hosted Google Photos replacement
6. Deploying CouchDB for Obsidian sync

<div style="display: flex; justify-content: center; align-items: center; flex-wrap: wrap;">
    <figure style="margin: 10px;">
        <img src="/images/homelab/Cluster-Overview.drawio.png" alt="Cluster Overview" width="300">
        <figcaption>Cluster Overview: 1CP + 2WP nodes with Apps on wp2</figcaption>
    </figure>
</div>

## Prerequisites

Before we begin, ensure you have:
- Completed [Part 1](/posts/2025/08/20/homelab-part1/) (Kubernetes cluster setup)
- Completed [Part 2](/posts/2025/09/05/homelab-part2/) (Core infrastructure and tools)
- A working ArgoCD installation with ApplicationSets configured
- Vault with External Secrets Operator for secret management
- An SMB or NFS share for persistent storage

You can find all the configurations in my [homelab repository](https://github.com/NovoG93/homelab).

## The App of Apps Pattern with ArgoCD

In Part 2, we set up ArgoCD with ApplicationSets that use a GitGenerator to automatically discover and deploy applications. This pattern allows us to add new applications simply by creating a folder with the necessary manifests and an `app.yaml` file.

### Repository Structure

```shell
homelab/
├── apps/                    # Application workloads (scheduled on wp2)
│   ├── couchdb/
│   ├── immich/
│   ├── netshoot/
│   └── postiz-app/
├── core/                    # Core infrastructure (all nodes)
│   ├── calico/
│   ├── metallb-system/
│   └── metrics-server/
├── tools/                   # Tools and operators (scheduled on wp1)
│   ├── argocd/
│   ├── cert-manager/
│   ├── cloudnative-pg/
│   ├── external-dns/
│   ├── external-secret-operator/
│   ├── kyverno/
│   ├── nginx-ingress/
│   ├── pihole/
│   ├── tailscale/
│   ├── vault/
│   └── wildcard-tls/
└── helmCharts/              # Cached helm charts for offline use
```

### Application Definition Pattern

Each application includes an `app.yaml` file that ArgoCD's ApplicationSet reads:

```yaml
# apps/immich/app.yaml
name: immich
path: apps/immich
namespace: immich
project: default
```

The ApplicationSet then creates an ArgoCD Application for each discovered `app.yaml`:

```yaml
# tools/argocd/appSets/apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: apps-appset
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - git:
        repoURL: https://github.com/NovoG93/homelab
        revision: main
        files:
          - path: "apps/**/app.yaml"
  template:
    metadata:
      name: "{% raw %}{{ .name }}{% endraw %}"
      labels:
        group: apps
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
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - PrunePropagationPolicy=foreground
          - ApplyOutOfSyncOnly=true
          - ServerSideApply=true
```

Note the key differences from the `tools-appset`:
- **Automated sync**: Apps have `automated.prune: true` and `selfHeal: true` for fully automated GitOps
- **Group label**: Apps are labeled with `group: apps` for easy filtering in the ArgoCD UI

## CloudNative-PG for PostgreSQL

Before deploying applications that need databases, we need to set up the CloudNative-PG operator. This operator manages PostgreSQL clusters as Kubernetes-native resources.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [tools/cloudnative-pg](https://github.com/NovoG93/homelab/tree/main/tools/cloudnative-pg)

<details markdown="1">
<summary>Show CloudNative-PG installation commands</summary>

```bash
mkdir -p tools/cloudnative-pg
pushd tools/cloudnative-pg
cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization

helmCharts:
- includeCRDs: true
  name: cloudnative-pg
  repo: https://cloudnative-pg.github.io/charts
  version: 0.24.0
  releaseName: cloudnative-pg
  namespace: cloudnative-pg
  valuesFile: values.yaml
EOF

cat << EOF > values.yaml
tolerations:
  - key: tools
    operator: Exists
    effect: NoSchedule

affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: tools
              operator: Exists
EOF
kustomize build . | kubectl apply -n cloudnative-pg -f -
popd
```
</details>

The operator runs on the tools node (`wp1`), while the actual PostgreSQL clusters (managed by the operator) will run on the apps node (`wp2`) alongside their applications.


## Tailscale Operator for Secure Remote Access

[Tailscale](https://tailscale.com/) provides a zero-config VPN that makes it easy to securely access your homelab from anywhere. The Tailscale Operator automatically creates Tailscale ingresses for your services.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [tools/tailscale](https://github.com/NovoG93/homelab/tree/main/tools/tailscale)

<details markdown="1">
<summary>Show Tailscale Operator installation commands</summary>

```bash
mkdir -p tools/tailscale
pushd tools/tailscale
cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization

resources:
- es-operator-oauth.yaml
- proxy-class-config.yaml

helmCharts:
- includeCRDs: true
  name: tailscale-operator
  repo: https://pkgs.tailscale.com/helmcharts
  version: 1.88.4
  releaseName: tailscale
  namespace: tailscale
  valuesFile: values.yaml
EOF

cat << EOF > values.yaml
installCRDs: true

operatorConfig:
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

ingressClass:
  enabled: true
EOF

# Create a ProxyClass to ensure Tailscale proxies run on the tools node
cat << EOF > proxy-class-config.yaml
apiVersion: tailscale.com/v1alpha1
kind: ProxyClass
metadata:
  name: tool-node-config
spec:
  statefulSet:
    pod:
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
kustomize build . | kubectl apply -n tailscale -f -
popd
```
</details>

### Automatic Tailscale Ingress with Kyverno

Instead of manually creating Tailscale ingresses for each service, I use a Kyverno ClusterPolicy that automatically generates them from VirtualServer resources:

```yaml
# tools/kyverno/clusterPolicies/create-tailscale-ingress-from-virtualserver.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: create-tailscale-ingress-from-virtualserver
spec:
  validationFailureAction: Enforce
  useServerSideApply: true
  rules:
  - name: create-tailscale-ingress-from-virtualserver
    skipBackgroundRequests: false
    match:
      any:
      - resources:
          kinds:
          - k8s.nginx.org/v1/VirtualServer
    exclude:
      any:
      - resources:
          kinds:
          - k8s.nginx.org/v1/VirtualServer
          selector:
            matchLabels:
              ignore-ts: "true"  # Add this label to skip Tailscale ingress
    generate:
      apiVersion: networking.k8s.io/v1
      kind: Ingress
      name: {% raw %}tailscale-{{request.object.metadata.name}}-ig{% endraw %}
      namespace: "{% raw %}{{request.object.metadata.namespace}}{% endraw %}"
      synchronize: true
      generateExisting: true
      data:
        metadata:
          annotations:
            created-by: kyverno.io/create-tailscale-ingress-from-virtualserver
            tailscale.com/proxy-class: "tool-node-config"
          labels: "{% raw %}{{ request.object.metadata.labels || parse_json('{}') }}{% endraw %}"
        spec:
          ingressClassName: tailscale
          tls:
            - hosts:
                - "{% raw %}{{request.object.metadata.name}}-{{request.object.metadata.namespace}}{% endraw %}"
          rules:
            - http:
                paths:
                  - path: /
                    pathType: Prefix
                    backend:
                      service:
                        name: "{% raw %}{{request.object.spec.upstreams[0].service}}{% endraw %}"
                        port:
                          number: "{% raw %}{{request.object.spec.upstreams[0].port}}{% endraw %}"
```

This policy:
1. Watches for any VirtualServer resource creation
2. Generates a corresponding Tailscale Ingress automatically
3. Uses the `tool-node-config` ProxyClass to ensure proxies run on the tools node
4. Can be skipped by adding the `ignore-ts: "true"` label to a VirtualServer

Now every application with a VirtualServer automatically gets both:
- **Local access** via nginx-ingress + Pi-hole DNS (e.g., `immich.novotny.live`)
- **Remote access** via Tailscale (e.g., `immich-immich` on your tailnet)


## Deploying Immich - Self-Hosted Google Photos

[Immich](https://immich.app/) is a self-hosted photo and video backup solution. It's the centerpiece of my homelab, replacing Google Photos with full local control.

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [apps/immich](https://github.com/NovoG93/homelab/tree/main/apps/immich)

### Architecture Overview

The Immich deployment consists of:
- **immich-server**: Main API server
- **immich-machine-learning**: ML server for face recognition and search
- **valkey**: Redis-compatible cache (replacement for Redis)
- **PostgreSQL**: Database managed by CloudNative-PG with vector extensions
- **SMB storage**: For the actual photo/video library

### PostgreSQL Cluster with Vector Extensions

Immich requires PostgreSQL with vector extensions for its ML features. Using CloudNative-PG:

```yaml
# apps/immich/cloudnative-pg/pg-db.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: immich-postgres
spec:
  instances: 1
  storage:
    pvcTemplate:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: nfs-client

  # Use the vectorchord image for vector similarity search
  imageName: ghcr.io/tensorchord/cloudnative-vectorchord:16.9-0.4.3

  postgresql:
    shared_preload_libraries:
      - "vchord.so"
  
  bootstrap:
    initdb:
      database: immich-db
      owner: immich
      secret:
        name: immich-database-credentials
      postInitApplicationSQL:
        # Required extensions for Immich
        - CREATE EXTENSION vchord CASCADE;
        - CREATE EXTENSION earthdistance CASCADE;

  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: apps
                operator: Exists
    tolerations:
      - key: apps
        operator: Exists
        effect: NoSchedule
```

### Managing Secrets with External Secrets Operator

Database credentials are stored in Vault and synced via ExternalSecret:

```yaml
# apps/immich/es-immich-postgres-user.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: immich-database-credentials
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: vault-dev
    kind: ClusterSecretStore
  target:
    name: immich-database-credentials
    deletionPolicy: Delete
    template:
      type: kubernetes.io/basic-auth
      data:
        username: "{% raw %}{{ .username }}{% endraw %}"
        password: "{% raw %}{{ .password }}{% endraw %}"
  data:
    - secretKey: username
      remoteRef:
        key: apps/immich/immich-postgres-credentials
        property: DB_USERNAME
    - secretKey: password
      remoteRef:
        key: apps/immich/immich-postgres-credentials
        property: DB_PASSWORD
```

### SMB Storage for Photo Library

Immich stores photos on an SMB share from my NAS:

```yaml
# apps/immich/immich-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: immich-smb
spec:
  capacity:
    storage: 500Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: smb
  mountOptions:
    - dir_mode=0777
    - file_mode=0777
  csi:
    driver: smb.csi.k8s.io
    volumeHandle: smb-server.default.svc.cluster.local/share##
    volumeAttributes:
      source: //192.168.0.173/immich
    nodeStageSecretRef:
      name: smb-creds
      namespace: immich
---
# apps/immich/immich-pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: immich-smb-claim
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Gi
  volumeName: immich-smb
  storageClassName: smb
```

### Helm Values for Immich

<details markdown="1">
<summary>Show Immich values.yaml</summary>

```yaml
# apps/immich/values.yaml
controllers:
  main:
    containers:
      main:
        image:
          tag: v2.0.1
        env:
          REDIS_HOSTNAME: '{% raw %}{{ printf "%s-valkey" .Release.Name }}{% endraw %}'
          DB_HOSTNAME: "immich-postgres-rw.immich.svc.cluster.local"
          DB_USERNAME:
            valueFrom:
              secretKeyRef:
                name: immich-database-credentials
                key: username
          DB_PASSWORD:
            valueFrom:
              secretKeyRef:
                name: immich-database-credentials
                key: password
          DB_DATABASE_NAME: immich-db

immich:
  persistence:
    library:
      existingClaim: immich-smb-claim
  configuration:
    trash:
      enabled: true
      days: 30
    storageTemplate:
      enabled: true
      template: "{% raw %}{{y}}/{{y}}-{{MM}}-{{dd}}/{{filename}}{% endraw %}"

server:
  enabled: true
  image:
    repository: ghcr.io/immich-app/immich-server
  ingress:
    main:
      enabled: false  # Using VirtualServer instead

machine-learning:
  enabled: true
  image:
    repository: ghcr.io/immich-app/immich-machine-learning
  persistence:
    cache:
      enabled: true
      size: 10Gi
      type: persistentVolumeClaim
      accessMode: ReadWriteMany
      storageClass: nfs-client

defaultPodOptions:
  tolerations:
    - key: apps
      operator: Exists
      effect: NoSchedule
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: apps
                operator: Exists

valkey:
  enabled: true
  controllers:
    main:
      containers:
        main:
          image:
            repository: docker.io/valkey/valkey
            tag: 8.1-alpine
```
</details>

### VirtualServer for Ingress

```yaml
# apps/immich/virtualserver.yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: immich
  annotations:
    external-dns.alpha.kubernetes.io/hostname: "immich.novotny.live"
  labels:
    tailscale.com/funnel: "false"
spec:
  externalDNS:
    enable: true
  host: immich.novotny.live
  tls:
    secret: wildcard-tls
  upstreams:
  - name: immich
    service: immich-server
    port: 2283
  routes:
  - path: /
    action:
      pass: immich
```

This VirtualServer:
1. Exposes Immich at `immich.novotny.live` on the local network
2. Uses the wildcard TLS certificate (copied by Kyverno)
3. Automatically creates a DNS record in Pi-hole via external-dns
4. Triggers Kyverno to create a Tailscale ingress for remote access


## Deploying CouchDB for Obsidian Sync

I use [Obsidian](https://obsidian.md/) for note-taking, and CouchDB provides a self-hosted sync solution via the [Obsidian LiveSync plugin](https://github.com/vrtmrz/obsidian-livesync).

[![YAML](https://img.shields.io/badge/YAML%20Files-CB171E?logo=yaml&logoColor=fff)](#) @ [apps/couchdb](https://github.com/NovoG93/homelab/tree/main/apps/couchdb)

<details markdown="1">
<summary>Show CouchDB installation commands</summary>

```bash
mkdir -p apps/couchdb
pushd apps/couchdb
cat << EOF > app.yaml
name: couchdb
path: apps/couchdb
namespace: couchdb
project: default
EOF

cat << EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: kustomization

resources:
- virtualserver.yaml
- es-obsidian-couchdb.yaml

helmCharts:
- includeCRDs: true
  name: couchdb
  repo: https://apache.github.io/couchdb-helm
  version: 4.6.1
  releaseName: obsidian
  namespace: couchdb
  valuesFile: values.yaml
EOF

cat << EOF > values.yaml
clusterSize: 1
allowAdminParty: false

autoSetup:
  enabled: true
  defaultDatabases:
    - _users
    - _replicator
    - _global_changes
    - georg_vault  # Personal vault database

createAdminSecret: false
adminUsernameKey: "adminUsername"
adminPasswordKey: "adminPassword"

persistentVolume:
  enabled: true
  size: 10Gi

image:
  repository: couchdb
  tag: 3.5.0

tolerations:
  - key: apps
    operator: Exists
    effect: NoSchedule

affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: apps
              operator: Exists

couchdbConfig:
  couchdb:
    uuid: decafbaddecafbaddecafbaddecafbad
    max_document_size: 104857600
  chttpd_auth:
    require_valid_user: true
  chttpd:
    bind_address: any
    require_valid_user: true
    max_http_request_size: 204857600
    enable_cors: true
  httpd:
    WWW-Authenticate: '"Basic realm="couchdb""'
    enable_cors: true
  cors:
    credentials: true
    origins: "app://obsidian.md,capacitor://localhost,http://localhost"
EOF
popd
```
</details>

Key configuration points:
- **CORS enabled**: Required for Obsidian to connect from desktop/mobile apps
- **Allowed origins**: Includes Obsidian's app protocols
- **Auto-setup**: Creates necessary system databases and a personal vault database
- **Credentials from Vault**: Admin credentials stored in Vault via ExternalSecret


## Verification

After deploying all applications, verify they're working:

```bash
# Check ArgoCD applications
kubectl get applications -n argocd

# Expected output:
# NAME       SYNC STATUS   HEALTH STATUS
# immich     Synced        Healthy
# couchdb    Synced        Healthy
# ...

# Check all pods in apps namespaces
kubectl get pods -n immich
kubectl get pods -n couchdb

# Verify DNS records in Pi-hole
kubectl exec -n pihole pihole-xxx -- cat /etc/pihole/hosts/custom.list | grep novotny
# 192.168.0.210 immich.novotny.live
# 192.168.0.210 couchdb.novotny.live

# Check Tailscale ingresses (auto-generated by Kyverno)
kubectl get ingress --all-namespaces -l created-by=kyverno.io/create-tailscale-ingress-from-virtualserver
```


## Summary

In this final part of the GitOps homelab series, we covered:

| Component | Purpose | Key Features |
|-----------|---------|--------------|
| **App of Apps Pattern** | Automated app discovery | GitGenerator scans for app.yaml files |
| **CloudNative-PG** | PostgreSQL operator | Kubernetes-native database management |
| **Tailscale Operator** | Secure remote access | Zero-config VPN with auto-provisioning |
| **Kyverno Policy** | Ingress automation | Auto-generates Tailscale ingress from VirtualServers |
| **Immich** | Photo management | Self-hosted Google Photos replacement |
| **CouchDB** | Obsidian sync | Self-hosted note synchronization |

### The Complete Picture

With all three parts completed, we now have:

1. **Kubernetes Cluster** (Part 1): kubeadm-bootstrapped with control plane and workers
2. **Infrastructure** (Part 2): Networking, storage, secrets, certificates, and GitOps
3. **Applications** (Part 3): Self-hosted services with automated deployment

The beauty of this setup is its maintainability:
- **Adding a new app**: Create a folder with manifests and `app.yaml` → ArgoCD deploys it
- **Updating an app**: Push to Git → ArgoCD syncs automatically
- **Secret management**: Add to Vault → ESO syncs to Kubernetes
- **Remote access**: Every VirtualServer automatically gets a Tailscale ingress

### What's Next?

Some ideas for extending this setup:
- **Monitoring**: Add Prometheus + Grafana for observability
- **Backup**: Implement Velero for cluster backup/restore
- **GPU Passthrough**: Add a GPU worker node for ML workloads (see my [k8s-setup repo](https://github.com/NovoG93/k8s-setup) for GPU passthrough documentation)
- **More Apps**: Jellyfin, Home Assistant, Nextcloud, etc.

I hope this series has been helpful in understanding how to build a production-grade Kubernetes homelab. Feel free to explore my [homelab repository](https://github.com/NovoG93/homelab) for the complete configurations!

---

**← [Part 1: Setting up the Kubernetes cluster](/posts/2025/08/20/homelab-part1/)** | **← [Part 2: Core Infrastructure and Tools](/posts/2025/09/05/homelab-part2/)**
