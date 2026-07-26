# Kubernetes Notes

Consolidated notes covering everything in the `kubernetes commands/` folder — basic kubectl,
config files, ConfigMaps/Secrets, volumes, networking, Ingress, private registries, Helm,
managed clusters, and Prometheus monitoring.

---

## Table of Contents

1. [What Kubernetes Does](#1-what-kubernetes-does)
2. [Cluster Architecture](#2-cluster-architecture)
3. [Core API Objects](#3-core-api-objects)
4. [Local Setup — Minikube + kubectl](#4-local-setup--minikube--kubectl)
5. [kubectl — Core Commands](#5-kubectl--core-commands)
6. [Configuration Files (YAML) Explained](#6-configuration-files-yaml-explained)
7. [Demo Project — MongoDB + Mongo Express](#7-demo-project--mongodb--mongo-express)
8. [ConfigMaps & Secrets — Env Vars vs Volumes](#8-configmaps--secrets--env-vars-vs-volumes)
9. [Namespaces](#9-namespaces)
10. [Volumes, PV, PVC & StorageClass](#10-volumes-pv-pvc--storageclass)
11. [Container Communication & Pod Networking](#11-container-communication--pod-networking)
12. [Ingress](#12-ingress)
13. [Pulling Images from a Private Registry](#13-pulling-images-from-a-private-registry)
14. [Helm — Package Manager for K8s](#14-helm--package-manager-for-k8s)
15. [Managed Cluster Demo — Linode LKE](#15-managed-cluster-demo--linode-lke)
16. [Monitoring — Prometheus Operator + Exporters](#16-monitoring--prometheus-operator--exporters)
17. [CI/CD — GitHub Actions](#17-cicd--github-actions)
18. [Troubleshooting](#18-troubleshooting)
19. [Quick Cheat-Sheet](#19-quick-cheat-sheet)

---

## 1. What Kubernetes Does

Kubernetes (K8s) is a container orchestration tool. It automates:

- Deploying containerized applications
- Scaling them up/down based on load
- Self-healing (restarting/replacing failed containers automatically)
- Load balancing traffic across running instances
- Rolling updates and rollbacks with zero downtime
- Managing configuration and secrets

**The core idea — declarative state.** You describe the *desired state* in YAML; K8s controllers
continuously compare it to the *actual state* and reconcile the difference. You never tell K8s
"start a container" — you tell it "2 replicas should exist" and it makes that true, forever.

### Alternatives

| Tool | Notes |
|---|---|
| **Docker Swarm** | Simpler, built into Docker, far less feature-rich |
| **HashiCorp Nomad** | General-purpose scheduler, not container-specific |

---

## 2. Cluster Architecture

A cluster is made of two kinds of machines:

- **Control plane node(s)** — the "brain" of the cluster
- **Worker nodes** — where your actual application containers run

### Control Plane Components

| Component | Role |
|---|---|
| `kube-apiserver` | Front door for the K8s API — all traffic (kubectl, internal components) goes through it |
| `etcd` | Key-value store holding all cluster data (the single source of truth) |
| `kube-scheduler` | Decides which node a new Pod lands on, based on resource availability |
| `kube-controller-manager` | Runs the control loops that reconcile actual state toward desired state |
| `cloud-controller-manager` | Same idea, for cloud-provider-specific integrations (load balancers, disks) |

### Worker Node Components

| Component | Role |
|---|---|
| `kubelet` | Agent on each node; starts/stops containers scheduled to that node |
| `kube-proxy` | Manages network rules on the node so Pods/Services are reachable |
| Container Runtime | Actually runs containers (containerd, CRI-O) |

---

## 3. Core API Objects

### Workload

- **Pod** — smallest deployable unit. Wraps one or more containers sharing network + storage
  context. You rarely create Pods directly in production — higher-level objects manage them.
- **Deployment** — manages **stateless** Pods. Handles scaling, rolling updates, rollbacks.
  Internally creates and manages a ReplicaSet.
  - **ReplicaSet** — keeps N identical Pods running. Managed by the Deployment; don't create directly.
- **StatefulSet** — like a Deployment but for **stateful** apps (databases). Gives each Pod a
  stable network identity (`mongodb-0`, `mongodb-1`, …) and its own persistent storage.
- **DaemonSet** — runs one Pod on *every* node (log collectors, node exporters).

### Storage

- **Volume** — storage attached to a Pod. Many types (local disk, NFS, cloud disks, ConfigMap, Secret).
- **PersistentVolume (PV)** — a cluster-level storage resource, provisioned independently of any Pod.
- **PersistentVolumeClaim (PVC)** — a Pod's *request* for storage; binds to a matching PV.
- **StorageClass** — defines *how* PVs get provisioned automatically (dynamic provisioning).

### Networking

- **Service** — stable network endpoint in front of a set of Pods. Pods are ephemeral and their
  IPs change; a Service gives a fixed DNS name + IP.
- **Ingress** — HTTP/HTTPS routing into the cluster, with host/path rules, in front of Services.

### Configuration

- **ConfigMap** — non-sensitive config as key-value pairs; injected as env vars or mounted files.
- **Secret** — same, but for sensitive data. **Base64-encoded, not encrypted by default.**

> Reference: [The Twelve-Factor App — Config](https://12factor.net/config) — keep config out of
> code. ConfigMaps/Secrets are how K8s implements this.

---

## 4. Local Setup — Minikube + kubectl

**Minikube** = a single-node K8s cluster on your machine (control plane + worker in one VM/container).
**kubectl** = the CLI you use to talk to *any* cluster.

Install:
- kubectl — https://kubernetes.io/docs/tasks/tools/
- Minikube — https://minikube.sigs.k8s.io/docs/start/

### Install on macOS (Homebrew)

```bash
brew update
brew install hyperkit
brew install minikube
```

### Shell autocompletion

```bash
eval "$( kubectl completion bash )"
eval "$( minikube completion bash )"
```

### Create the cluster

```bash
minikube start                            # uses the default driver (docker) — recommended
minikube start --vm-driver=hyperkit       # explicit hyperkit driver (macOS)
minikube start --cpus 4 --memory 8192     # bigger cluster, needed for Prometheus stack
```

### Verify

```bash
kubectl get nodes
minikube status
kubectl version
```

### Delete & restart in debug mode

```bash
minikube delete
minikube start --vm-driver=hyperkit --v=7 --alsologtostderr
minikube status
```

`--v=7 --alsologtostderr` raises log verbosity and prints to the terminal — use it when the
cluster refuses to start.

### Other minikube commands

```bash
minikube service <service-name>   # open a NodePort service in the browser (creates a tunnel)
minikube ip                       # cluster IP — needed for /etc/hosts entries with Ingress
minikube ssh                      # shell into the minikube node itself
minikube addons list
minikube addons enable ingress    # installs the NGINX Ingress Controller
```

> **Driver tip:** if Pods won't start under `hyperkit`, just run plain `minikube start` to use
> the Docker driver. This fixes most mongo/mongo-express startup failures.

---

## 5. kubectl — Core Commands

### Cluster & Nodes

```bash
kubectl version
kubectl cluster-info
kubectl get nodes
kubectl describe node <node-name>
```

### Pods

```bash
kubectl get pod
kubectl get pod -o wide            # adds node + Pod IP columns
kubectl get pod --watch            # live-updating view
kubectl describe pod <pod-name>    # events + config — the #1 debugging command
kubectl logs <pod-name>
kubectl logs -f <pod-name>                    # follow live
kubectl logs <pod-name> -c <container-name>   # specific container in a multi-container Pod
kubectl exec -it <pod-name> -- /bin/bash      # shell into a container
```

### Deployments

```bash
kubectl create deployment nginx-depl --image=nginx
kubectl get deployment
kubectl get replicaset
kubectl edit deployment nginx-depl          # opens live config in $EDITOR; saving applies it
kubectl delete deployment nginx-depl
kubectl scale deployment <name> --replicas=5

kubectl rollout status deployment <name>
kubectl rollout history deployment <name>
kubectl rollout undo deployment <name>      # roll back to previous revision
```

### Services

```bash
kubectl get services
kubectl get svc                       # shorthand
kubectl describe service <name>
```

Check `Endpoints:` in `describe service` — **if it's empty, your selector doesn't match any Pod labels.**
That's the most common Service bug.

### ConfigMaps & Secrets

```bash
kubectl get configmap
kubectl get secret
kubectl describe secret <name>
```

### Apply / Delete configuration files

```bash
kubectl apply -f <file>.yaml     # create or update — idempotent
kubectl delete -f <file>.yaml
kubectl apply -f ./dir/          # apply every manifest in a directory
```

### Metrics

```bash
kubectl top node
kubectl top pod
```

`kubectl top` returns current CPU and memory usage for nodes or Pods. Requires the metrics-server
addon (`minikube addons enable metrics-server`).

### General / Useful

```bash
kubectl get all                     # overview of most resources in the current namespace
kubectl get all | grep <keyword>
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl api-resources               # every resource type the cluster knows about
kubectl edit <resource> <name>
kubectl delete <resource> <name>
```

---

## 6. Configuration Files (YAML) Explained

Every K8s manifest has **3 parts**:

| Part | Who writes it | Meaning |
|---|---|---|
| `metadata` | you | name, namespace, labels — identity |
| `spec` | you | the **desired state** |
| `status` | Kubernetes | the **actual state**, auto-generated — never write this |

K8s stores the whole object in etcd and the controller loop drives `status` toward `spec`.

### Deployment example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:                 # <-- Pod blueprint: has its own metadata + spec
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.16
        ports:
        - containerPort: 8080
```

**Key fields**

- `apiVersion` — API group + version (`apps/v1` for Deployment, `v1` for Pod/Service/ConfigMap/Secret)
- `kind` — the resource type
- `spec.replicas` — how many Pod copies to maintain
- `spec.selector.matchLabels` — which Pods this Deployment owns. **Must match `spec.template.metadata.labels`.**
- `spec.template` — the Pod blueprint (a nested metadata + spec)

### Service example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx          # matches Pod labels
  ports:
    - protocol: TCP
      port: 80          # port the Service exposes
      targetPort: 8080  # port on the container — must equal containerPort
```

**Service types**

| Type | Reachable from | Use for |
|---|---|---|
| `ClusterIP` (default) | inside the cluster only | databases, internal APIs |
| `NodePort` | `<nodeIP>:<30000-32767>` | quick external access in dev |
| `LoadBalancer` | external IP from the cloud provider | production external access |

### How Deployments and Services connect

**Labels are the only glue.** There is no direct name reference — a Service's `spec.selector`
must match the labels on the Pods (set via `template.metadata.labels`). Three things must line up:

```
Deployment.spec.selector.matchLabels  ==  Deployment.spec.template.metadata.labels  ==  Service.spec.selector
Service.spec.ports.targetPort         ==  container.ports.containerPort
```

### What K8s adds — the "result" file

Applying the short manifest above and running `kubectl get deployment nginx-deployment -o yaml`
returns a much larger object. K8s fills in defaults and live state:

```yaml
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment", ...}     # your original manifest, verbatim
  creationTimestamp: "2020-01-24T10:54:56Z"
  resourceVersion: "96574"
  uid: e1075fa3-6468-43d0-83c0-63fede0dae51
spec:
  progressDeadlineSeconds: 600
  revisionHistoryLimit: 10
  strategy:                       # <-- defaulted for you
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    spec:
      containers:
      - imagePullPolicy: IfNotPresent
        terminationMessagePath: /dev/termination-log
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
status:                            # <-- entirely generated
  availableReplicas: 2
  readyReplicas: 2
  replicas: 2
  updatedReplicas: 2
  conditions:
  - type: Available
    status: "True"
    reason: MinimumReplicasAvailable
```

Two things worth knowing:

- `last-applied-configuration` is how `kubectl apply` computes a 3-way diff — it remembers what
  *you* declared vs what the server defaulted, so re-applying doesn't wipe defaults.
- `strategy.rollingUpdate` defaults to 25%/25%, which is *why* rolling updates are zero-downtime
  without you configuring anything.

### Workflow

```bash
vim nginx-deployment.yaml
kubectl apply -f nginx-deployment.yaml
kubectl get pod
kubectl get deployment
kubectl delete -f nginx-deployment.yaml
```

---

## 7. Demo Project — MongoDB + Mongo Express

A realistic multi-component setup: **Secret → Database → ConfigMap → Frontend**.

### Architecture

```
Browser
   │  (NodePort 30000 / LoadBalancer)
   ▼
mongo-express-service  ──►  mongo-express Pod
                                   │  reads DB host from ConfigMap
                                   │  reads credentials from Secret
                                   ▼
                          mongodb-service (ClusterIP)  ──►  mongodb Pod
```

### 1. Secret — `mongo-secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
    name: mongodb-secret
type: Opaque
data:
    mongo-root-username: dXNlcm5hbWU=     # base64 of "username"
    mongo-root-password: cGFzc3dvcmQ=     # base64 of "password"
```

Generate the values:

```bash
echo -n 'username' | base64
echo -n 'password' | base64      # -n matters: no trailing newline
```

> **Base64 is encoding, not encryption.** Anyone with read access to the Secret can decode it.
> Real protection needs RBAC + encryption at rest (or an external vault).

### 2. Database — `mongo.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deployment
  labels:
    app: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-password
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  selector:
    app: mongodb
  ports:
    - protocol: TCP
      port: 27017
      targetPort: 27017
```

No `type:` → defaults to **ClusterIP**, which is correct: the database should only be reachable
from inside the cluster.

> `---` separates multiple objects in one YAML file. Grouping tightly-coupled objects
> (Deployment + its Service) in one file is a common convention.

### 3. ConfigMap — `mongo-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-configmap
data:
  database_url: mongodb-service      # the Service name IS the DNS hostname
```

Inside the cluster, `mongodb-service` resolves via CoreDNS. Full form:
`mongodb-service.default.svc.cluster.local`.

### 4. Frontend — `mongo-express.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-express
  labels:
    app: mongo-express
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-express
  template:
    metadata:
      labels:
        app: mongo-express
    spec:
      containers:
      - name: mongo-express
        image: mongo-express
        ports:
        - containerPort: 8081
        env:
        - name: ME_CONFIG_MONGODB_ADMINUSERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username
        - name: ME_CONFIG_MONGODB_ADMINPASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-password
        - name: ME_CONFIG_MONGODB_SERVER
          valueFrom:
            configMapKeyRef:
              name: mongodb-configmap
              key: database_url
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-express-service
spec:
  selector:
    app: mongo-express
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 8081
      targetPort: 8081
      nodePort: 30000       # only valid for NodePort/LoadBalancer; range 30000–32767
```

### Deployment order

```bash
kubectl apply -f mongo-secret.yaml
kubectl apply -f mongo.yaml
kubectl apply -f mongo-configmap.yaml
kubectl apply -f mongo-express.yaml
```

**Why order matters:** a Pod referencing a Secret or ConfigMap that doesn't exist yet gets stuck
in `CreateContainerConfigError` and will not start. Create config objects *before* the workloads
that consume them.

### Verify

```bash
kubectl get pod
kubectl get pod --watch
kubectl get pod -o wide
kubectl get service
kubectl get secret
kubectl get all | grep mongodb
```

### Debug

```bash
kubectl describe pod mongodb-deployment-xxxxxx
kubectl describe service mongodb-service
kubectl logs mongo-express-xxxxxx
```

### Access it

```bash
minikube service mongo-express-service
```

Minikube opens a tunnel and launches the browser. (On minikube, `LoadBalancer` behaves like
`NodePort` — there's no cloud provider to hand out a real external IP.)

### Concepts this demo reinforces

- **Secret** → DB credentials, shared by *both* Deployments via `secretKeyRef`
- **ConfigMap** → internal DB hostname via `configMapKeyRef`
- **ClusterIP Service** → MongoDB, internal only
- **External Service** → Mongo Express, reachable by a human

---

## 8. ConfigMaps & Secrets — Env Vars vs Volumes

Two ways to consume a ConfigMap/Secret. **Which one you need depends on what the app expects.**

| | Env var (`valueFrom`) | Volume mount |
|---|---|---|
| Good for | single values: hostname, port, username | whole config **files**: `mosquitto.conf`, `nginx.conf`, certs |
| Updates | **frozen** at Pod start — needs a restart | file contents auto-update (with a delay) |
| Key becomes | an env var | a **file name** inside `mountPath` |

### Env var style

```yaml
env:
- name: MONGO_INITDB_ROOT_USERNAME
  valueFrom:
    secretKeyRef:              # configMapKeyRef for a ConfigMap
      name: mongodb-secret
      key: username
```

Load *every* key at once with `envFrom`:

```yaml
envFrom:
- configMapRef:
    name: mongodb-configmap
- secretRef:
    name: mongodb-secret
```

### Volume style — `mosquitto-config-components.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mosquitto-config-file
data:
  mosquitto.conf: |          # the "|" block scalar preserves newlines -> a real file
    log_dest stdout
    log_type all
    log_timestamp true
    listener 9001
---
apiVersion: v1
kind: Secret
metadata:
  name: mosquitto-secret-file
type: Opaque
data:
  secret.file: |
    c29tZXN1cGVyc2VjcmV0IGZpbGUgY29udGVudHMgbm9ib2R5IHNob3VsZCBzZWU=
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mosquitto
  labels:
    app: mosquitto
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mosquitto
  template:
    metadata:
      labels:
        app: mosquitto
    spec:
      containers:
        - name: mosquitto
          image: eclipse-mosquitto:1.6.2
          ports:
            - containerPort: 1883
          volumeMounts:
            - name: mosquitto-conf
              mountPath: /mosquitto/config
            - name: mosquitto-secret
              mountPath: /mosquitto/secret
              readOnly: true
      volumes:
        - name: mosquitto-conf
          configMap:
            name: mosquitto-config-file
        - name: mosquitto-secret
          secret:
            secretName: mosquitto-secret-file
```

Result inside the container:

```
/mosquitto/config/mosquitto.conf     <- key name became the file name
/mosquitto/secret/secret.file        <- decoded automatically; you never see base64
```

**Two-step pattern, always the same:**
1. `spec.template.spec.volumes` — declare the volume and point it at the ConfigMap/Secret
2. `containers[].volumeMounts` — mount it at a path inside the container

Note the `data:` key differences: `configMap` uses `name:`, but `secret` uses `secretName:`.

> The source file also contains a second, plain `mosquitto` Deployment with no volumes — that's
> the "before" version from the tutorial. Both share the name `mosquitto`, so applying the whole
> file leaves you with whichever came last (the plain one). Split them, or delete the second block.

### Creating them imperatively

```bash
kubectl create configmap my-config --from-file=mosquitto.conf
kubectl create configmap my-config --from-literal=db_host=mongodb-service
kubectl create secret generic my-secret --from-literal=password=mypwd
```

---

## 9. Namespaces

Namespaces organize/isolate resources inside one cluster (`dev`, `staging`, `prod`, or per team).

```bash
kubectl get namespaces
kubectl create namespace <name>
kubectl get pods -n <namespace>
kubectl get pods -A                                        # --all-namespaces
kubectl config set-context --current --namespace=<name>    # change the default
```

Default namespaces: `default`, `kube-system` (control plane components), `kube-public`,
`kube-node-lease`.

> Most `kubectl` commands target the `default` namespace unless you pass `-n`. This is the
> single most common reason resources look "missing."

**Cross-namespace DNS:** a Service in another namespace is reached as
`<service>.<namespace>` (e.g. `mongodb-service.database`). ConfigMaps and Secrets are **not**
shared across namespaces — you must create a copy in each.

---

## 10. Volumes, PV, PVC & StorageClass

Container filesystems are ephemeral — restart a Pod, lose the data. Volumes fix that.

### The three-piece model

```
Pod ──mounts──► PVC ──binds to──► PV ──backed by──► real storage (NFS, EBS, local disk)
                 │
                 └──may be auto-created by──► StorageClass (dynamic provisioning)
```

- **PV** — the actual storage resource. Admin-provisioned, cluster-scoped, outlives Pods.
- **PVC** — "I need 10Gi, ReadWriteOnce." Namespaced. K8s finds/creates a matching PV.
- **StorageClass** — a template for creating PVs on demand, so nobody hand-writes PVs.

**Why the split:** developers request storage (PVC) without knowing whether it's AWS EBS, NFS, or
a local SSD. The PV/StorageClass hides the backend.

### Ephemeral volume types

- **emptyDir** — scratch space created when the Pod starts, deleted when the Pod dies. Good for
  cache or sharing files between containers in the same Pod.
- **hostPath** — mounts a path from the node's filesystem. Avoid in multi-node clusters — data is
  tied to one node and doesn't move with the Pod.

### PersistentVolume — `persistent-volumes.yaml`

NFS-backed:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-name
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.0
  nfs:
    path: /dir/path/on/nfs/server
    server: nfs-server-ip-address
```

Google Cloud persistent disk:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-volume
  labels:
    failure-domain.beta.kubernetes.io/zone: us-central1-a__us-central1-b
spec:
  capacity:
    storage: 400Gi
  accessModes:
  - ReadWriteOnce
  gcePersistentDisk:
    pdName: my-data-disk
    fsType: ext4
```

Local disk with node affinity:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:                  # required for local volumes — pins Pods to the node
    required:                    # that physically has the disk
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - example-node
```

The `spec` key (`nfs:`, `gcePersistentDisk:`, `local:`) is what makes each PV a different
storage backend — everything else is identical.

**Access modes**

| Mode | Meaning |
|---|---|
| `ReadWriteOnce` (RWO) | read-write by a single **node** |
| `ReadOnlyMany` (ROX) | read-only by many nodes |
| `ReadWriteMany` (RWX) | read-write by many nodes (NFS, CephFS — not most cloud block storage) |

**Reclaim policies** — what happens to the PV when its PVC is deleted:

| Policy | Effect |
|---|---|
| `Retain` | Keep the data; the PV must be reclaimed manually. Safest. |
| `Delete` | Delete the underlying storage too. Default for dynamic provisioning. |
| `Recycle` | Deprecated — basic scrub then reuse. |

### PersistentVolumeClaim — `persistent-volume-claims.yaml`

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-name
spec:
  storageClassName: manual
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
     name: mypvc
spec:
     accessModes:
     - ReadWriteOnce
     resources:
       requests:
         storage: 100Gi
     storageClassName: storage-class-name    # -> dynamic provisioning
```

A PVC binds to a PV only if **capacity, accessModes, and storageClassName all match**. If nothing
matches and no StorageClass applies, the PVC sits in `Pending` — and any Pod using it stays
`Pending` too.

### StorageClass — `storage-class.yaml`

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: storage-class-name
provisioner: kubernetes.io/aws-ebs     # the plugin that creates real storage
parameters:
  type: io1
  iopsPerGB: "10"
  fsType: ext4
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
     name: mypvc
spec:
     accessModes:
     - ReadWriteOnce
     resources:
       requests:
         storage: 100Gi
     storageClassName: storage-class-name
```

With a StorageClass, the PVC triggers **dynamic provisioning** — the provisioner creates the EBS
volume and PV automatically. No hand-written PVs.

```bash
kubectl get storageclass
kubectl get pv
kubectl get pvc
```

### Using volumes in a Pod — `pods-with-volume.yaml`

PVC-backed:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: pvc-name
```

ConfigMap-backed:

```yaml
  volumes:
    - name: config-dir
      configMap:
        name: bb-configmap
```

Secret-backed:

```yaml
  volumes:
  - name: secret-dir
    secret:
      secretName: bb-secret
```

Same two-step pattern every time: declare under `volumes`, mount under `volumeMounts`. The
`name` is the link between them.

### Multiple volumes in one Deployment — `deployment-with-multiple-volumes.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elastic
spec:
  selector:
    matchLabels:
      app: elastic
  template:
    metadata:
      labels:
        app: elastic
    spec:
      containers:
      - image: elastic:latest
        name: elastic-container
        ports:
        - containerPort: 9200
        volumeMounts:
        - name: es-persistent-storage
          mountPath: /var/lib/data
        - name: es-secret-dir
          mountPath: /var/lib/secret
        - name: es-config-dir
          mountPath: /var/lib/config
      volumes:
      - name: es-persistent-storage
        persistentVolumeClaim:
          claimName: es-pv-claim
      - name: es-secret-dir
        secret:
          secretName: es-secret
      - name: es-config-dir
        configMap:
          name: es-config-map
```

Three different volume *types* in one Pod — persistent data, credentials, and config — each at
its own mount path. This is the standard shape for a real stateful application.

> Note: in a Deployment, `volumes` sits at `spec.template.spec.volumes` (inside the Pod template),
> not at `spec.volumes`. A frequent mistake.

---

## 11. Container Communication & Pod Networking

### The Pod networking model

- **Every Pod gets its own IP address**, flat across the cluster — no NAT between Pods.
- **Containers in the same Pod share one network namespace.** They reach each other over
  `localhost` and cannot both bind the same port.
- Pod IPs are **ephemeral** — never hardcode one. Use a Service name.

So there are two distinct communication paths:

| Path | Mechanism |
|---|---|
| Container ↔ container, same Pod | `localhost:<port>` |
| Pod ↔ Pod (different Pods) | Service DNS name → `kube-proxy` load-balances to a Pod IP |

### Single-container Pod — `postgres.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  containers:
  - name: postgres
    image: postgres:9.6.17
    ports:
    - containerPort: 5432
    env:
    - name: POSTGRES_PASSWORD
      value: "pwd"       # plain value — fine for a demo, use a Secret in reality
```

### Sidecar pattern — `nginx-sidecar-container.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx-container
    image: nginx
    ports:
    - containerPort: 80
  - name: sidecar
    image: curlimages/curl
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the sidecar container; sleep 300"]
```

Two containers, one Pod. The sidecar can hit the main container at `localhost:80` — no Service
needed. Real uses: log shippers, metrics exporters, proxies (service mesh), config reloaders.

Test it:

```bash
kubectl exec -it nginx -c sidecar -- /bin/sh
# inside:
curl localhost:80          # reaches nginx-container
```

`-c <container>` selects which container in a multi-container Pod — required for `exec` and
`logs` here.

**Sidecar vs separate Pod:** use a sidecar when the two containers must scale together and share
a lifecycle/filesystem. Otherwise use separate Deployments + a Service.

### DNS resolution

| From | Address |
|---|---|
| Same namespace | `mongodb-service` |
| Other namespace | `mongodb-service.database` |
| Fully qualified | `mongodb-service.database.svc.cluster.local` |

---

## 12. Ingress

Ingress manages external HTTP/HTTPS routing into the cluster, sitting in front of Services.

**Why not just NodePort/LoadBalancer:**
- One entry point for many services, with host- and path-based routing rules
- SSL/TLS termination in one place
- Avoids paying for a separate cloud LoadBalancer (and IP) per service

### Two moving parts

1. **Ingress resource** — the routing *rules* (just data in etcd)
2. **Ingress Controller** — a Pod (usually NGINX) that *reads* those rules and actually proxies traffic

**Without a controller running, an Ingress resource does nothing.**

```bash
minikube addons enable ingress          # installs NGINX Ingress Controller
kubectl get pods -n ingress-nginx       # confirm it's running
```

### Example — `dashboard-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: kubernetes-dashboard      # must be in the same namespace as the target Service
spec:
  ingressClassName: "nginx"
  rules:
  - host: dashboard.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kubernetes-dashboard
            port:
              number: 80
```

### Example — path with `Exact`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: name
  annotations:
    kubernetes.io/ingress.class: "nginx"     # older style — prefer ingressClassName
spec:
  rules:
    - host: app.com
      http:
        paths:
        - path: /
          pathType: Exact
          backend:
            service:
              name: my-service
              port:
                number: 8080
```

**Key fields**

- `host` — the domain the request must carry (matched against the HTTP `Host` header)
- `path` + `pathType` — `Prefix` (matches `/` and everything under it) or `Exact`
- `backend.service` — which Service and port to forward to
- `ingressClassName` — which controller handles this rule. **Replaces the deprecated
  `kubernetes.io/ingress.class` annotation** — use `ingressClassName` on `networking.k8s.io/v1`.

### Making a fake host resolve locally

```bash
minikube ip                # e.g. 192.168.49.2
sudo vim /etc/hosts        # Windows: C:\Windows\System32\drivers\etc\hosts
# add:
192.168.49.2  dashboard.com
```

Then browse to `http://dashboard.com`.

```bash
kubectl get ingress
kubectl get ingress -n kubernetes-dashboard
kubectl describe ingress dashboard-ingress -n kubernetes-dashboard
```

An `ADDRESS` column that stays empty usually means no controller is running, or
`ingressClassName` doesn't match an installed IngressClass.

### API version history — worth knowing

| Version | Status | Backend syntax |
|---|---|---|
| `extensions/v1beta1` | **removed in K8s 1.22** | `serviceName` / `servicePort` |
| `networking.k8s.io/v1beta1` | deprecated | `serviceName` / `servicePort` |
| `networking.k8s.io/v1` | **current** | `service.name` / `service.port.number` |

The `linode-kubernetes-engine-demo/test-ingress.yaml` file uses the old removed form:

```yaml
apiVersion: extensions/v1beta1     # will be REJECTED on K8s 1.22+
...
            backend:
              serviceName: mongo-express-service
              servicePort: 8081
```

Modern equivalent:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mongo-express
spec:
  ingressClassName: nginx
  rules:
    - host: nb-139-162-140-213.frankfurt.nodebalancer.linode.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: mongo-express-service
                port:
                  number: 8081
```

---

## 13. Pulling Images from a Private Registry

By default the kubelet can only pull public images. For a private repo it needs credentials,
supplied as a Secret of type `kubernetes.io/dockerconfigjson`.

### Step 1 — log in to the registry

```bash
docker login -u username -p password
aws ecr get-login                     # prints the full docker login command for AWS ECR
```

This writes credentials to `~/.docker/config.json`:

```bash
cat .docker/config.json
cat .docker/config.json | base64
```

### Step 2 — create the Secret

From the generated config file:

```bash
kubectl create secret generic my-registry-key \
  --from-file=.dockerconfigjson=.docker/config.json \
  --type=kubernetes.io/dockerconfigjson
```

Or directly from credentials (no `docker login` needed):

```bash
kubectl create secret docker-registry my-registry-key \
  --docker-server=https://private-repo \
  --docker-username=user \
  --docker-password=pwd
```

```bash
kubectl get secret
```

Declarative equivalent — `docker-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-registry-key
data:
  .dockerconfigjson: base64-encoded-contents-of-.docker/config.json-file
type: kubernetes.io/dockerconfigjson
```

### Step 3 — reference it with `imagePullSecrets` — `my-app-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      imagePullSecrets:            # <-- Pod-level, sibling of containers
      - name: my-registry-key
      containers:
      - name: my-app
        image: privat-repo/my-app:1.3
        imagePullPolicy: Always
        ports:
          - containerPort: 3000
```

**`imagePullSecrets` sits at the Pod spec level, not inside a container.** It's the kubelet that
pulls images, not the container.

`imagePullPolicy` values: `Always`, `IfNotPresent` (default), `Never`. Use `Always` when you
overwrite a mutable tag like `:latest`.

### Getting the config from inside minikube

If you ran `docker login` inside the minikube node:

```bash
minikube ssh                                                      # shell into the node
scp -i $(minikube ssh-key) docker@$(minikube ip):.docker/config.json .docker/config.json
```

> The Secret must live in the **same namespace** as the Pod that uses it.

**Symptom of a missing/wrong pull secret:** Pod stuck in `ErrImagePull` / `ImagePullBackOff`.
Confirm with `kubectl describe pod <name>` and read the Events section.

---

## 14. Helm — Package Manager for K8s

Helm bundles YAML manifests into reusable, versioned, parameterized **charts**.

**Why:** deploying MongoDB by hand means writing StatefulSet + Service + Secret + PVC + headless
Service. A chart does it in one command, and lets you override just the bits you care about.

### Repos

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://charts.helm.sh/stable
helm repo update
helm search repo mongodb
```

> The old `https://kubernetes-charts.storage.googleapis.com/` URL is **dead**. Use
> `https://charts.helm.sh/stable` instead. (The `prometheus-exporter/commands.md` file still has
> the old URL — the `setup-prometheus-operator/commands.md` one is correct.)

### Install / upgrade / remove

```bash
helm install <release-name> <chart>
helm install prometheus prometheus-community/kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack --version "9.4.1"   # pin it

helm install mongodb bitnami/mongodb -f values.yaml     # override defaults
helm install mongodb bitnami/mongodb --set replicaCount=3

helm ls
helm upgrade <release-name> <chart>
helm uninstall <release-name>
```

**Release name** = your instance of the chart. It prefixes all created resources
(`prometheus-grafana`, `prometheus-kube-prometheus-prometheus`) — which is why the
port-forward commands below look so verbose.

**Pin versions in anything real.** `helm install` without `--version` grabs the newest chart,
so the same command can produce different results next month.

### values files

A chart's `values.yaml` holds all defaults; you override only what you need:

```yaml
# test-mongodb-values.yaml
architecture: replicaset
replicaCount: 3
persistence:
    storageClass: "linode-block-storage"
auth:
    rootPassword: secret-root-pwd
```

```yaml
# prometheus-exporter values.yaml
mongodb:
  uri: "mongodb://mongodb-service:27017"

serviceMonitor:
  additionalLabels:
    release: prometheus
```

```bash
helm show values <chart>        # see everything you *could* override
```

---

## 15. Managed Cluster Demo — Linode LKE

Deploying the same app on a real managed cluster instead of minikube.

### Flow

1. Create the cluster in the Linode dashboard (LKE), pick node count/size
2. Download the `kubeconfig` YAML
3. Point kubectl at it:

```bash
export KUBECONFIG=~/Downloads/mongodb-kubeconfig.yaml
kubectl get nodes
```

4. Deploy MongoDB as a **replicaset** via Helm, using cloud block storage:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install mongodb --values test-mongodb-values.yaml bitnami/mongodb
kubectl get pods
kubectl get pvc                # block storage volumes created automatically
kubectl get statefulset
```

`test-mongodb-values.yaml`:

```yaml
architecture: replicaset
replicaCount: 3
persistence:
    storageClass: "linode-block-storage"
auth:
    rootPassword: secret-root-pwd
```

5. Deploy Mongo Express — `test-mongo-express.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-express
  labels:
    app: mongo-express
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-express
  template:
    metadata:
      labels:
        app: mongo-express
    spec:
      containers:
      - name: mongo-express
        image: mongo-express
        ports:
        - containerPort: 8081
        env:
        - name: ME_CONFIG_MONGODB_ADMINUSERNAME
          value: root
        - name: ME_CONFIG_MONGODB_SERVER
          value: mongodb-0.mongodb-headless      # <-- StatefulSet Pod DNS
        - name: ME_CONFIG_MONGODB_ADMINPASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb                       # Secret created by the Helm chart
              key: mongodb-root-password
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-express-service
spec:
  selector:
    app: mongo-express
  ports:
    - protocol: TCP
      port: 8081
      targetPort: 8081
```

**The interesting bit:** `mongodb-0.mongodb-headless`. A StatefulSet gives each Pod a stable
ordinal name (`mongodb-0`, `-1`, `-2`) and a **headless Service** (`clusterIP: None`) gives each
one an individual DNS record. That's how you address the *primary* replica specifically —
something a normal Deployment + Service cannot do.

Note it also consumes the Secret the Helm chart generated — inspect chart-created objects with
`kubectl get secret` before writing your own.

6. Expose it with Ingress (Linode NodeBalancer as the entry point) — see
   [§12](#12-ingress) for the modern `networking.k8s.io/v1` rewrite of `test-ingress.yaml`.

### Minikube vs managed cluster

| | Minikube | Managed (LKE/EKS/GKE) |
|---|---|---|
| Nodes | 1 | many, real |
| `LoadBalancer` | behaves like NodePort | provisions a real cloud LB + public IP |
| StorageClass | `standard` (hostPath) | cloud block storage, real persistence |
| kubeconfig | auto-configured | download and `export KUBECONFIG` |

---

## 16. Monitoring — Prometheus Operator + Exporters

### Why an Operator

Prometheus alone needs a config file, Alertmanager, Grafana, node-exporter, kube-state-metrics,
and dashboards. The **kube-prometheus-stack** chart installs all of it and adds CRDs
(`ServiceMonitor`, `PrometheusRule`) so you configure monitoring with normal K8s YAML instead of
hand-editing `prometheus.yml`.

### Cluster with enough headroom

```bash
minikube start --cpus 4 --memory 8192 --vm-driver hyperkit
```

The stack is heavy — a default minikube will struggle.

### Install the stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://charts.helm.sh/stable
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack --version "9.4.1"
```

Chart: https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack

```bash
kubectl get pods
kubectl get all
kubectl get crd            # ServiceMonitor, PrometheusRule, Alertmanager, ...
```

### Access the UIs (port-forward)

```bash
kubectl port-forward service/prometheus-kube-prometheus-prometheus 9090
kubectl port-forward svc/prometheus-kube-prometheus-alertmanager 9093
kubectl port-forward deployment/prometheus-grafana 3000
```

Grafana login:

```
user: admin
pwd:  prom-operator      # chart default, from values.yaml
```

`kubectl port-forward` tunnels a cluster Service to `localhost` without exposing it publicly —
the right way to reach admin UIs. It runs in the foreground; Ctrl-C ends it.

### Monitoring your own app — the MongoDB exporter

Prometheus scrapes HTTP metrics endpoints. MongoDB doesn't speak Prometheus, so you run an
**exporter** that translates its stats.

Target app — `mongodb.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deployment
  labels:
    app: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo
        ports:
        - containerPort: 27017
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  selector:
    app: mongodb
  ports:
    - protocol: TCP
      port: 27017
      targetPort: 27017
```

Exporter config — `values.yaml`:

```yaml
mongodb:
  uri: "mongodb://mongodb-service:27017"

serviceMonitor:
  additionalLabels:
    release: prometheus
```

Install:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install mongodb-exporter prometheus-community/prometheus-mongodb-exporter
helm install mongodb-exporter prometheus-community/prometheus-mongodb-exporter --version "2.8.1"
helm install mongodb-exporter prometheus-community/prometheus-mongodb-exporter -f values.yaml
```

Chart: https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-mongodb-exporter

```bash
kubectl port-forward service/mongodb-exporter-prometheus-mongodb-exporter 9216
```

Then open `localhost:9216/metrics` to see the raw metrics.

> **`release: prometheus` is the critical line.** The kube-prometheus-stack only picks up
> ServiceMonitors carrying the label matching its own release name. Get it wrong and the exporter
> runs fine but never appears in Prometheus. Check Prometheus UI → **Status → Targets**.

### The full chain

```
MongoDB Pod  ──►  mongodb-exporter  ──►  ServiceMonitor  ──►  Prometheus  ──►  Grafana
  (app)          (translates to           (tells Prom          (scrapes &      (dashboards)
                  Prometheus format)       what to scrape)      stores)
                                                  │
                                                  └──►  Alertmanager  ──►  email/Slack
```

---

## 17. CI/CD — GitHub Actions

Workflows live in `.github/workflows/*.yaml` and run on repo events (push, PR, release).

Reference links:

- Official actions — https://github.com/actions
- Marketplace — https://github.com/marketplace?type=actions
- Events that trigger workflows — https://docs.github.com/en/actions/reference/events-that-trigger-workflows
- Docker build & push action — https://github.com/marketplace/actions/docker-build-push

Typical K8s pipeline: **build image → push to registry → `kubectl apply` / `helm upgrade`**.
The private-registry Secret from [§13](#13-pulling-images-from-a-private-registry) is what lets
the cluster pull the image the pipeline just pushed.

---

## 18. Troubleshooting

### Command sequence for any broken Pod

```bash
kubectl get pod                                       # 1. what's the STATUS?
kubectl describe pod <pod-name>                       # 2. read the Events at the bottom
kubectl logs <pod-name>                               # 3. what did the app say?
kubectl logs <pod-name> --previous                    #    logs from the crashed instance
kubectl get events --sort-by=.metadata.creationTimestamp
```

`describe` explains why K8s can't run it; `logs` explains why the app itself failed.

### Pod status → likely cause

| Status | Cause | Fix |
|---|---|---|
| `Pending` | No node has the resources, or a PVC is unbound | `describe pod`; check `kubectl get pvc` |
| `ImagePullBackOff` / `ErrImagePull` | Wrong image name/tag, or missing pull secret | Verify the name; see [§13](#13-pulling-images-from-a-private-registry) |
| `CreateContainerConfigError` | Referenced ConfigMap/Secret doesn't exist | Create it first; check the namespace |
| `CrashLoopBackOff` | Container starts then exits | `kubectl logs --previous` — it's an app error |
| `RunContainerError` | Bad command/entrypoint or mount | `describe pod` |
| `Terminating` (stuck) | Finalizer or slow shutdown | `kubectl delete pod <name> --force --grace-period=0` |

### mongo-express / mongo-db Pods won't start

Use the **docker driver** — run plain `minikube start` without specifying `--vm-driver`.
Installation guide: https://minikube.sigs.k8s.io/docs/start/

### Service exists but nothing responds

```bash
kubectl describe service <name>     # is "Endpoints:" empty?
```

Empty endpoints = the Service `selector` doesn't match any Pod labels. Compare:

```bash
kubectl get pods --show-labels
```

Also verify `targetPort` equals the container's `containerPort`.

### Resource "missing"

Almost always the wrong namespace:

```bash
kubectl get <resource> -A
```

### Ingress not routing

1. Is a controller running? `kubectl get pods -n ingress-nginx`
2. Does `ingressClassName` match? `kubectl get ingressclass`
3. Does the Ingress live in the **same namespace** as the target Service?
4. Does `/etc/hosts` point the host at `minikube ip`?

### PVC stuck in `Pending`

```bash
kubectl get pvc
kubectl describe pvc <name>
kubectl get pv
kubectl get storageclass
```

No matching PV and no StorageClass → nothing to bind to. Check that capacity, `accessModes`,
and `storageClassName` all agree.

### Cluster won't start

```bash
minikube delete
minikube start --v=7 --alsologtostderr
```

---

## 19. Quick Cheat-Sheet

```bash
# ---- Cluster ----
minikube start
minikube start --cpus 4 --memory 8192
minikube status
minikube ip
minikube delete
minikube addons enable ingress
kubectl cluster-info
kubectl get nodes

# ---- Pods ----
kubectl get pod
kubectl get pod -o wide
kubectl get pod --watch
kubectl describe pod <name>
kubectl logs <name>
kubectl logs -f <name>
kubectl logs <name> --previous
kubectl logs <name> -c <container>
kubectl exec -it <name> -- /bin/bash

# ---- Deployments ----
kubectl create deployment <name> --image=<image>
kubectl get deployment
kubectl get replicaset
kubectl edit deployment <name>
kubectl scale deployment <name> --replicas=3
kubectl rollout status deployment <name>
kubectl rollout history deployment <name>
kubectl rollout undo deployment <name>
kubectl delete deployment <name>

# ---- Services & Ingress ----
kubectl get svc
kubectl describe service <name>
kubectl get ingress
minikube service <service-name>
kubectl port-forward svc/<name> <port>

# ---- Config ----
kubectl get configmap
kubectl get secret
kubectl describe secret <name>
kubectl create configmap <name> --from-file=<file>
kubectl create configmap <name> --from-literal=key=value
kubectl create secret generic <name> --from-literal=key=value
kubectl create secret docker-registry <name> \
  --docker-server=<url> --docker-username=<u> --docker-password=<p>
echo -n 'value' | base64

# ---- Storage ----
kubectl get pv
kubectl get pvc
kubectl get storageclass

# ---- Apply / Delete ----
kubectl apply -f <file>.yaml
kubectl apply -f ./dir/
kubectl delete -f <file>.yaml

# ---- Namespaces ----
kubectl get namespaces
kubectl create namespace <name>
kubectl get pods -n <namespace>
kubectl get pods -A
kubectl config set-context --current --namespace=<name>

# ---- Helm ----
helm repo add <name> <url>
helm repo update
helm search repo <chart>
helm show values <chart>
helm install <release> <chart> -f values.yaml
helm install <release> <chart> --version "x.y.z"
helm ls
helm upgrade <release> <chart>
helm uninstall <release>

# ---- Metrics & Debugging ----
kubectl top node
kubectl top pod
kubectl get all
kubectl get all | grep <keyword>
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl api-resources
kubectl describe <resource> <name>
```

---

## Reference Links

| Topic | Link |
|---|---|
| kubectl install | https://kubernetes.io/docs/tasks/tools/ |
| Minikube start guide | https://minikube.sigs.k8s.io/docs/start/ |
| Helm | https://helm.sh |
| k3s (lightweight K8s) | https://k3s.io |
| Lens (cluster GUI) | https://k8slens.dev |
| kube-prometheus-stack chart | https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack |
| prometheus-mongodb-exporter chart | https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-mongodb-exporter |
| GitHub Actions marketplace | https://github.com/marketplace?type=actions |
| Twelve-Factor Config | https://12factor.net/config |
