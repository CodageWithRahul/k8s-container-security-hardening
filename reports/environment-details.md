# Environment details

## Operating system and tooling

| Item | Value | Evidence |
|---|---|---|
| OS | Microsoft Windows 10 Enterprise (build 10.0.26200.9711) | `cmd /c ver` |
| Docker Desktop | 29.9.2 | `docker --version` |
| Minikube | v1.39.0 | `minikube version` |
| kubectl client | v1.37.2 | `kubectl version --client --output=yaml` |
| Kubernetes server | v1.37.0 | `kubectl version --output=yaml` |
| Trivy | 0.76.2 | `trivy --version` |
| kube-bench | Official Aqua image `v0.16.0` available via Docker; no standalone host binary | `docker run --rm docker.io/aquasec/kube-bench:v0.16.0 version` returned `v0.16.0` |
| Kubernetes context | `minikube` | `kubectl config current-context` |
| Container runtime | Docker Desktop / containerd-backed Minikube runtime | `kubectl version` and local cluster state |
| Network plugin | Calico | `kubectl -n kube-system get ds -o wide` |

## Active cluster state

`kubectl get ns` shows the following namespaces:

- `default`
- `kube-system`
- `kube-public`
- `kube-node-lease`
- `kubernetes-dashboard`
- `secure-app`

The `secure-app` namespace is active and contains the hardened demo application and its network policy set.

## Network plugin note

The cluster is using `calico-node` in the `kube-system` namespace, which supports NetworkPolicy enforcement. This was confirmed by the active DaemonSet output and by successful allowed/denied traffic validation during pod-to-pod tests.

## CIS benchmark limitation

Docker Desktop and Minikube are running, but no final CIS result was generated.
The official `aquasec/kube-bench:v0.16.0` image contains `cis-1.12` and does
not contain a profile for Kubernetes 1.37. Its versioned support matrix maps
CIS 1.12 to Kubernetes 1.32-1.34. The current main-branch documentation lists
CIS 2.0 only through Kubernetes 1.35, still below the active cluster's 1.37.

Aqua's official node Job also uses `hostPID: true` and read-only mounts of
sensitive node directories, including `/etc/kubernetes`, `/var/lib/etcd`, and
`/var/lib/kubelet`. That access was not approved, so the Job was not applied.
The Minikube node also lacks `/etc/kubernetes/pki`, and `kubelet` is not in its
command path, which may affect generic file/process checks.

The exact probe evidence is in `scans/kube-bench/final.txt`; it is not a final
benchmark result. No unsupported or stale profile was run against Kubernetes
1.37.

## Security conclusion

The local environment supports safe validation of application-layer RBAC and
NetworkPolicy controls. A supported kube-bench profile for Kubernetes 1.37 and
explicit approval for the official Job's host PID and read-only host-path
access are required before a final node/control-plane benchmark can be run.
