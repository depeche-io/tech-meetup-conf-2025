
Prepare a support slide deck in Markdown (Marp).
For each Kubernetes related topic prepare 4 slides (~3 mins of presentation) with in the following template:
* Feature overview slide
  * 1-2 sentence summary
  * Current state in version 1.34 (alpha, beta, stable, ...)
  * Research planned next step in 1.35 upcoming version in k8s
  * Find relevant KEPs numbers (Kubernetes Enahncement Proposals on GitHub)
* What it solves / Example of a problem before
* Example of new feature
* Why it matters - intended use-cases with examples


Use YAML examples, images links or generated where possible.


* Online Pod resizing, enabling you to modify CPU and memory settings on the fly without restarting Pods in-place pod vertical scaling
* Pod-level resource allocation provides granular control by setting resource constraints on a per-pod basis, ensuring fair usage and easier troubleshooting
* VolumeSnapshots and GroupVolumeSnapshots simplify backup and restore processes across multiple volumes
* OCI (Open Container Initiative) image and artifact volumes (ImageVolumes)
* SWAP support (Java Application use-case) https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2400-node-swap#enable-swap-support-only-for-burstable-qos-pods
* Job Success and Failure Policies (Stable)
* https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/



---

What's next

- Gang scheduling
- KEP-3521	Node Log Query to Beta
