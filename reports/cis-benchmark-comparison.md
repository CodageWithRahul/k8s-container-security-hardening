# CIS benchmark comparison

## Baseline evidence

The historical output is preserved in `scans/kube-bench/baseline.txt` and
`scans/kube-bench/baseline-output.log`. The counted totals in the baseline are:

| Result | Count |
|---|---:|
| PASS | 56 |
| FAIL | 14 |
| WARN | 61 |
| INFO | 21 |

The baseline does not identify its kube-bench release or benchmark version, so
it cannot be compared one-to-one with a later run unless that provenance is
recovered.

## Environment and official image verification

The active environment was checked without modifying the cluster:

| Component | Observed value |
|---|---|
| Docker Desktop engine | 29.9.2, Linux (`docker-desktop`) |
| Minikube | v1.39.0, running |
| Minikube driver | Docker |
| Container runtime | containerd |
| Kubernetes server | v1.37.0 |
| Node OS | Debian GNU/Linux 12 |

The official Aqua image was pulled and its CLI was executed without privileges,
host mounts, or cluster access:

```text
Image: docker.io/aquasec/kube-bench:v0.16.0
Command: docker run --rm docker.io/aquasec/kube-bench:v0.16.0 version
Output: v0.16.0
Digest: sha256:75506f222d1eb6ce2a751a5533bdc0a3b54c898e2e49e7751d0ee22cfb862679
```

Inspection of the image's bundled configuration directories confirmed that
`cis-1.12` is present, while `cis-2.0` and a Kubernetes 1.37 profile are absent.
The versioned [Aqua support matrix](https://github.com/aquasecurity/kube-bench/blob/v0.16.0/docs/platforms.md)
maps CIS 1.12 to Kubernetes 1.32-1.34. Thus the official released image is not
documented for the current 1.37 cluster. The current main-branch support matrix
lists CIS 2.0 for Kubernetes 1.34-1.35, which still does not include 1.37; that
profile is also not in the inspected v0.16.0 image.

## Official execution method and access implications

Aqua documents running the benchmark in a Kubernetes Job. Its
[official v0.16.0 job manifest](https://github.com/aquasecurity/kube-bench/blob/v0.16.0/job.yaml)
uses `hostPID: true` and read-only `hostPath` mounts for paths including
`/etc/kubernetes`, `/var/lib/etcd`, `/var/lib/kubelet`, `/usr/bin`, and CNI
directories. It does not set `privileged: true`, but read-only access still
exposes sensitive control-plane configuration, etcd database contents, and
kubelet state to the container; `hostPID` also exposes host process metadata.
The official
[running guide](https://github.com/aquasecurity/kube-bench/blob/v0.16.0/docs/running.md)
describes the need for host PID and host configuration access.

The Minikube node path check found `/etc/kubernetes`, `/var/lib/etcd`,
`/var/lib/kubelet`, `/etc/cni/net.d`, and `/opt/cni/bin`; `/etc/kubernetes/pki`
was absent, and `kubelet` was not found in the node command path. These
Minikube-specific paths can make some generic checks inapplicable or cause
misleading file-check results.

Direct `docker run` on this Windows host would inspect the Docker Desktop Linux
environment, not the Minikube node's filesystem. It cannot provide a valid
node-level audit of the active cluster.

## Final benchmark status

No final CIS checks were executed. Therefore final PASS, FAIL, WARN, INFO, and
not-applicable counts are **NOT AVAILABLE**, not zero. The probe evidence and
exact limitations are recorded in `scans/kube-bench/final.txt`.

| Check group | Status | Reason |
|---|---|---|
| Docker availability and official image CLI | PASS | Docker daemon is healthy; official Aqua image returned version `v0.16.0`. |
| Profile compatibility with active Kubernetes | BLOCKED | The released image profile set does not support Kubernetes 1.37. |
| Node/control-plane benchmark | NOT RUN | Official Job's hostPID and sensitive read-only host mounts require explicit approval; none was given. |
| Baseline comparison | PARTIALLY COMPLETE | Historical totals exist, but baseline version provenance is missing and no compatible final run exists. |

## Safest practical alternative

Do not downgrade or reconfigure the active Minikube cluster. For a valid audit
of this cluster, wait for an official kube-bench profile that documents
Kubernetes 1.37 support, then review the Job's access and obtain approval
before applying it. A separate disposable Minikube cluster on a version
supported by the selected official image can be audited without disturbing the
active cluster, but its results must be labeled as belonging to that separate
cluster, not as the final result for Kubernetes 1.37.

No host-level remediations were performed.
