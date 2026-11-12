# Container Security and Kubernetes Hardening

## Project overview

This repository contains a local Minikube hardening project for a sample NGINX deployment in the `secure-app` namespace. It covers:

- least-privilege RBAC
- default-deny ingress controls
- DNS-only egress baseline
- workload security contexts
- Trivy image and manifest scanning
- baseline kube-bench evidence

The current implementation is intentionally conservative and preserves the original baseline results for comparison. It does not claim a full production-grade platform hardening state if the required benchmark tooling is unavailable or host-level control-plane settings are outside the safe repository scope.

## Repository layout

```text
.
├── README.md
├── manifests/
│   ├── rbac/
│   │   └── rbac.yaml
│   ├── network-policies/
│   │   ├── 00-cluster-export-evidence.yaml
│   │   ├── 01-default-deny.yaml
│   │   ├── 02-allow-dns.yaml
│   │   ├── 03-allow-demo-ingress.yaml
│   │   └── 04-allow-nettest-egress.yaml
│   └── workloads/
│       ├── 00-demo-original.yaml
│       ├── 00-secure-app-namespace.yaml
│       ├── 01-demo-hardened.yaml
│       ├── 02-demo-service.yaml
│       └── 03-nettest.yaml
├── scans/
│   ├── trivy/
│   │   ├── config-final.txt
│   │   ├── config-hardened.txt
│   │   ├── config-original.txt
│   │   ├── image-baseline.json
│   │   ├── image-baseline.txt
│   │   ├── image-final.json
│   │   ├── image-final.txt
│   │   ├── network-policies-final.txt
│   │   └── nettest-final.txt
│   └── kube-bench/
│       ├── baseline-output.log
│       ├── baseline.txt
│       └── final.txt
├── logs/
│   └── remediation-log.md
├── reports/
│   ├── cluster-audit-baseline.md
│   ├── completion-audit.md
│   ├── cis-benchmark-comparison.md
│   ├── environment-details.md
│   ├── final-security-report.docx
│   ├── final-security-report.md
│   ├── network-policy-validation.md
│   ├── project-scope.md
│   ├── rbac-baseline.txt
│   ├── rbac-validation.md
│   ├── rbac-verification.txt
│   ├── validation-evidence.md
│   ├── vulnerability-remediation.xlsx
│   └── network-policy-baseline.txt
└── .gitignore
```

## Prerequisites

- Windows 10 or 11
- Docker Desktop
- Minikube
- kubectl
- Trivy
- Git
- kube-bench in a supported Linux environment only when a final CIS run is required

## Quick start (CMD-compatible)

Use these commands at the repository root in Command Prompt.

1. Start the cluster and confirm the context:

```cmd
minikube start --driver=docker
kubectl config current-context
kubectl get nodes
```

2. Create the namespace:

```cmd
kubectl apply -f manifests/workloads/00-secure-app-namespace.yaml
```

3. Apply RBAC, policies, and the hardened workload:

```cmd
kubectl apply -f manifests/rbac/
kubectl apply -f manifests/network-policies/01-default-deny.yaml
kubectl apply -f manifests/network-policies/02-allow-dns.yaml
kubectl apply -f manifests/network-policies/03-allow-demo-ingress.yaml
kubectl apply -f manifests/network-policies/04-allow-nettest-egress.yaml
kubectl apply -f manifests/workloads/01-demo-hardened.yaml
kubectl apply -f manifests/workloads/02-demo-service.yaml
kubectl rollout status deployment/demo-app -n secure-app
```

4. Validate the workload and service:

```cmd
kubectl -n secure-app get pods,services,endpoints -o wide
kubectl -n secure-app get networkpolicies
```

5. Run the labeled network test client:

```cmd
kubectl apply -f manifests/workloads/03-nettest.yaml
kubectl -n secure-app exec nettest -- wget -qO- -T 3 http://demo-app:8080
kubectl -n secure-app label pod nettest role- --overwrite
kubectl -n secure-app exec nettest -- wget -qO- -T 3 http://demo-app:8080
kubectl -n secure-app delete pod nettest --ignore-not-found=true
```

In the current cluster, the first request succeeds and the second one times out after the label is removed.

6. Validate RBAC least privilege:

```cmd
kubectl auth can-i get pods --as=system:serviceaccount:secure-app:app-service-account -n secure-app
kubectl auth can-i delete pods --as=system:serviceaccount:secure-app:app-service-account -n secure-app
```

Expected result: `yes` for `get pods`, `no` for `delete pods`.

7. Run Trivy:

```cmd
trivy image --severity HIGH,CRITICAL --output scans/trivy/image-final.txt docker.io/nginxinc/nginx-unprivileged:mainline-alpine@sha256:b5047b9b76845eac124358373c6852377c52e4b05834a47647e955dd91cb8f1e
trivy config --severity MEDIUM,HIGH,CRITICAL --output scans/trivy/config-final.txt manifests/workloads/01-demo-hardened.yaml
trivy config --severity MEDIUM,HIGH,CRITICAL --output scans/trivy/network-policies-final.txt manifests/network-policies/
```

8. CIS benchmark status:

The official Aqua image was verified with Docker, but the released `v0.16.0`
image does not include a profile for this cluster's Kubernetes v1.37.0. Aqua's
versioned support matrix for that release lists Kubernetes 1.32-1.34 as the
newest supported range. The official Kubernetes Job also uses `hostPID: true`
and read-only mounts of sensitive node paths such as `/etc/kubernetes`,
`/var/lib/etcd`, and `/var/lib/kubelet`; do not apply it without reviewing the
exposure and obtaining approval. The exact investigation is recorded in
`scans/kube-bench/final.txt` and `reports/cis-benchmark-comparison.md`.

Do not use `docker run` directly against Docker Desktop as a substitute: that
would inspect the Docker Desktop Linux environment rather than the Minikube
node. No final benchmark counts are available for the active v1.37 cluster.

## Verified status

| Area | Current status |
|---|---|
| RBAC | Verified in the live cluster. `get` allowed, `delete` denied. |
| Network policy enforcement | Verified in the live cluster. Allowed client succeeds; unlabeled client times out. |
| Hardened workload | Present and running in the `secure-app` namespace. |
| Image scan | Final Trivy image report exists; the image still has unresolved vulnerabilities. |
| Manifest scan | Final config scan shows one medium registry trust finding (KSV-0125). |
| kube-bench final run | Blocked: official released profile does not support Kubernetes 1.37; official host PID/host-path access was not approved. |

## Documentation and evidence

- `reports/validation-evidence.md` contains the actual cluster observations and command outputs captured for the project.
- `reports/environment-details.md` documents the actual host and cluster environment.
- `reports/completion-audit.md` tracks requirement completion against the repo state.
- `logs/remediation-log.md` records open and closed findings.
- `reports/final-security-report.docx` and `reports/final-security-report.md` provide the final summary report.
- `reports/Industry_Project_Report.docx` is the comprehensive institute-format project report; fill in the student, company, institute, and actual-effort placeholders before submission.
- `reports/Industry_Project_Report.md` is its editable Markdown source.
