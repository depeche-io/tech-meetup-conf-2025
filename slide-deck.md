---
marp: true
theme: gaia
#header: The Gems of Kubernetes' Latest Features
#footer: David Pech @ Wrike | TechMeetup Conference 2025
class: [gaia]
#paginate: true
lang: en-US
transition: none
style: |
  section {
    align-content: start;
  }
  section::after {
    content: attr(data-marpit-pagination) ' / ' attr(data-marpit-pagination-total);
  }

---

TODO: nektere slidy precuhuji

# The Gems of Kubernetes' Latest Features

![Intro](./intro.png)

---

# David Pech

TODO: ksicht, format
![Wrike](./wrike.png)
![Sestra Emmy](./emmy.png)
![Golden Kubestronaut](./golden-kubestronaut.png)

---

# Why bother learning about that hyped K-word?

"So you can have the same features as before on VMs."

... but of course this time *smarter*.

---

# Online Pod Resizing

**In-place vertical scaling without restarts**
- Status: **Beta** in Kubernetes 1.34
- Next: Moving toward **Stable** in 1.35 TODO
- KEP: [1287](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/1287-in-place-update-pod-resources)

---

# The Problem Before

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-hungry-app
spec:
  containers:
  - name: app
    image: my-app:v1
    resources:
      limits:
        memory: "2Gi" # <-- restart
        cpu: "1000m" # <-- restart
```

---

# And Now

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resizable-app
spec:
  containers:
  - name: app
    image: my-app:v1
    resources:
      limits:
        memory: "4Gi" # CHANGED
        cpu: "2000m" # CHANGED
    resizePolicy:
    - { resourceName: memory, restartPolicy: NotRequired }
    - { resourceName: cpu, restartPolicy: NotRequired }
```

---

# Why Online Pod Resizing Matters

**Use Cases:**
- **Databases**: Scale memory up for heavy query periods
- **Auto-scaling**: Faster response to load changes

**But:**
- You probably don't care about CPU .limits
- Memory can scale only up
- Java

---

# Pod-level Resource Allocation

**Simplify with per-pod resource constraints**
- Status: **Beta** since Kubernetes 1.34
- KEP: [2837](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/2837-pod-level-resource-spec/README.md)

---

# The Problem Before

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database-pod
spec:
  containers:
  - name: postgres
    image: postgres:18
    resources:
      requests:
        cpu: "2000m"
        memory: "4Gi"
      limits:
        cpu: "2000m"
        memory: "4Gi"
  - name: exporter
    image: postgres_exporter
    resources: # <-- ???
      requests:
        cpu: "20m"
        memory: "4Mi"
      limits:
        cpu: "200m"
        memory: "40Mi"
```

---

# And Now

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database-pod
spec:
  resources: # Pod-level definition
    requests:
      cpu: "2000m"
      memory: "4Gi"
    limits:
      cpu: "2000m"
      memory: "4Gi"
  containers:
  - name: postgres
    image: postgres:18
  - name: exporter
    image: postgres_exporter
```

---

# Why Pod-level Allocation Matters

**Use Cases:**
- Sidecars
- (Uneven containers)

**Benefits:** Easier to understand, Less waste, QoS class

**But:** Some tooling not ready yet, No in-place resize

---

# VolumeSnapshots & VolumeGroupSnapshot

**Simplified backup and restore across multiple volumes**
- Status: VolumeSnapshots **Stable**, VolumeGroupSnapshot **Beta** in 1.34
- Next: VolumeGroupSnapshot moving to **Stable** in 1.35 (?)
- KEP: [3476](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/3476-volume-group-snapshot)

---

# The Problem Before

```bash
# Manual, error-prone backup process
kubectl exec mysql-pod -- mysqldump > backup.sql
kubectl cp mysql-pod:backup.sql ./local-backup.sql

# No coordination between related volumes
# Risk of inconsistent state across multiple volumes
# Manual restore process
```

---

# And Now

```yaml
# Snapshot for single PVC
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-snapshot-20241001
spec:
  volumeSnapshotClassName: csi-snapshotter
  source:
    persistentVolumeClaimName: mysql-pvc
# --> real data managed by VolumeSnapshotContent
```

---

# And Now

```yaml
# Multiple PVCs at the same time (multiple "mounts")
apiVersion: groupsnapshot.storage.k8s.io/v1beta1
kind: VolumeGroupSnapshot
metadata:
  name: app-group-snapshot
spec:
  volumeGroupSnapshotClassName: csi-group-snapshotter
  source:
    selector:
      matchLabels:
        app: database-cluster
# --> generates VolumeGroupSnapshotContent
```

---

# Why VolumeSnapshots Matter

![benchmark.png height:400px width:1000px](./benchmark.png)

[Source (Kubecon NA, Chicago)](https://kccncna2023.sched.com/event/1R2ml/disaster-recovery-with-very-large-postgres-databases-gabriele-bartolini-edb-michelle-au-google)

---

# Why VolumeSnapshots Matter

**Use Cases:**
- **Database Backups**: Consistent point-in-time snapshots
- **Disaster Recovery**: Quick restoration from snapshots
- **Development/Testing**: Clone production data safely
- **Multi-volume Applications**: Coordinated snapshots across volumes

---

# OCI ImageVolumes

**Reduce image duplication and network overhead**
- Status: **Beta** in Kubernetes 1.34
- KEP: [4639](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/4639-oci-volume-source)

---

# The Problem Before

```Dockerfile
# TODO: here
FROM XXX: as model

FROM myapp:latest AS app

#TODO: 
# enrich my app with LLM model
ADD --from=model model-file /mnt/model-file
```

```bash
docker build -t myapp:latest-with-model
```

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: my-fancy-model-including-app
    image: myapp:latest-with-model
```

---

# The Problem Before

Container design - either lightweight + limited:

- CloudNativePG

Or bulky:

- StackGres (100+ extensions)

---

# The Problem Before

```yaml
apiVersion: v1
kind: Pod
spec:
  initContainers:
    # download the dependency here
  containers:
  - name: my-fancy-model-including-app
    volumeMounts:
    - name: model-file
      mountPath: /mnt/
      readOnly: true
```

---

# And Now

```yaml
apiVersion: v1
kind: Pod
spec:
  volumes:
  - name: model-image
    image:
      reference: myregistry/large-base:v1
      pullPolicy: IfNotPresent
  containers:
  - name: my-fancy-model-including-app
    volumeMounts:
    - name: model-file
      mountPath: /mnt/
      readOnly: true
```

---

# Why OCI ImageVolumes Matter

**Use Cases:**
- **ML Workloads**: Share large model files across containers
- **Microservices**: Common base files and dependencies
- **Addons/Extensions**: Provide as regular image
- **Edge Computing**: Minimize bandwidth usage

---

# Node SWAP Support

**Handle memory spikes with flexible memory management**
- Status: **Beta** in Kubernetes 1.34 (Burstable QoS only)
- Next: Enhanced controls and monitoring in 1.35
- KEP: [2400](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2400-node-swap)

(Off by default)

---

# The Problem Before

No SWAP support.

---

# And Now

```yaml
# this fragment goes into the kubelet's configuration file
memorySwap:
  swapBehavior: LimitedSwap
---
# Only non-high-priority Pods under the Burstable QoS tier are permitted to use swap.
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: java-app
    image: openjdk:11
    resources:
      requests:
        memory: "2Gi"
      limits:
        memory: "4Gi"
```

---

# Why SWAP Support Matters

**Use Cases:**
- **Applications with Dormant Memory**: Java, batch operations, ...
- **Cost optimization**
- **You are in charge** and responsible for your choices

---

# Job Success and Failure Policies

**Better job management with fine-grained control**
- Status: **Stable** in Kubernetes 1.34
- Next: Enhanced observability features in 1.35
- KEP: [3329](https://github.com/kubernetes/enhancements/tree/master/keps/sig-apps/3329-retriable-and-counting-job-failures)

---

# The Problem Before

```yaml
# Simple job with basic retry logic
apiVersion: batch/v1
kind: Job
spec:
  backoffLimit: 3  # Retry 3 times, that's it
  template:
    spec:
      containers:
      - name: batch-processor
        image: batch-app:v1
      restartPolicy: Never
```

**Issues:** All-or-nothing retry, no failure categorization, poor observability

---

# Job Success and Failure Policies in Action

```yaml
apiVersion: batch/v1
kind: Job
spec:
  backoffLimit: 5
  podFailurePolicy:
    rules:
    - action: FailJob
      onExitCodes:
        containerName: batch-processor
        operator: In
        values: [1, 2]  # Fatal errors - don't retry
    - action: Ignore
      onPodConditions:
      - type: DisruptionTarget  # Node maintenance - retry
  successPolicy:
    rules:
    - succeededIndexes: "0-9"
      succeededCount: 8  # 80% success rate is enough
```

---

# Why Job Success and Failure Policies Matter

**Use Cases:**
- **Data Processing**: Skip corrupted data, continue with rest
- **ML Training**: Handle node failures gracefully
- **ETL Pipelines**: Categorize transient vs permanent failures
- **Batch Analytics**: Partial success scenarios

**Benefits:**
- Intelligent failure handling
- Reduced unnecessary retries
- Better resource utilization
- Improved job completion rates

---

# What's Next in Kubernetes

**Upcoming features to watch:**

- **Gang Scheduling** - Coordinate scheduling of related pods
  - KEP: [2716](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/2716-pod-group-api)
  - Status: Alpha development for 1.35

- **Node Log Query** - Query node logs through API
  - KEP: [3521](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/3521-node-log-query)
  - Status: Moving to Beta in 1.35

---

# Key Takeaways

 **Kubernetes is quietly evolving** with powerful new capabilities

 **Stateful workloads** are becoming first-class citizens

 **Resource management** is getting more sophisticated

 **Operational complexity** is being reduced through automation

**Start experimenting** with these features in your development environments!

---

# Thank You!

**Questions?**

**David Pech**
*Solution Architect @ Wrike*

**Slides:** Available at this repository
**Contact:** Connect on LinkedIn or at the conference

*Let's bring more workloads to Kubernetes! *
