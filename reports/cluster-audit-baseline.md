# Cluster audit baseline report

## Environment summary

- Windows 10 build 10.0.26200.9711
- Docker Desktop 29.9.2
- Minikube 1.39.0
- Kubernetes server 1.37.0
- kubectl client 1.37.2
- Trivy 0.76.2
- kube-bench: official Aqua image `v0.16.0` verified through Docker; no compatible current-version profile
- Cluster context: `minikube`

## Baseline benchmark results

`scans/kube-bench/baseline.txt` contains the retained baseline output, and the actual summary is:

- PASS: 56
- FAIL: 14
- WARN: 61
- INFO: 21

The failures and warnings are concentrated in host-level control-plane checks related to:

- etcd data directory permissions and ownership
- API server audit logging configuration
- kubelet certificate authority settings
- profiling options on the API server, controller-manager, and scheduler
- host file permissions for control-plane PKI assets and CNI files

## Important limitations

- This is a local Minikube deployment, not a production cluster.
- The official `v0.16.0` image's CIS 1.12 profile is documented for Kubernetes 1.32-1.34; the active cluster is Kubernetes 1.37. The current main-branch docs list CIS 2.0 only through Kubernetes 1.35.
- The official node-scanning Job requires `hostPID: true` and read-only mounts of sensitive node directories. It was not applied because explicit approval was not given.
- The Minikube node lacks `/etc/kubernetes/pki`, and `kubelet` is not in its command path; some generic path/process checks may not map to this node.
- Node and control-plane changes are outside the repository's workload-level configuration scope.
- The project intentionally preserves historical baseline evidence without overwriting it.

## Assessment

The baseline identifies valid security concerns for a Kubernetes control plane, but there is no compatible final benchmark for the active Kubernetes 1.37 cluster. The official image and documented method were investigated; no unsupported profile was run and no sensitive host paths were mounted. See `reports/cis-benchmark-comparison.md` and `scans/kube-bench/final.txt` for exact evidence and the safest next step.
