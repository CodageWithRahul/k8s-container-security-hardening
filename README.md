# Container Security and Kubernetes Hardening

**Securing a Modern Orchestration Platform**

## Project overview

This TCS iON project assesses and hardens a sample application running on
Kubernetes. It covers container image vulnerabilities, Kubernetes RBAC,
network isolation, Pod Security Standards (PSS), and CIS Kubernetes Benchmark
checks. The project is scoped to a local, single-node Minikube cluster; it is
not a production deployment.

The manifests apply least-privilege RBAC, default-deny ingress, DNS-only
baseline egress, and narrow rules for a labeled test client to reach the demo
application. The hardened workload uses a non-root identity, a read-only root
filesystem, dropped capabilities, resource limits, health checks, and a
digest-pinned unprivileged NGINX image.

## Architecture and repository layout

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
│       ├── 00-secure-app-namespace.yaml
│       ├── 00-demo-original.yaml
│       ├── 01-demo-hardened.yaml
│       ├── 02-demo-service.yaml
│       └── 03-nettest.yaml
├── scans/
│   ├── trivy/
│   │   ├── config-original.txt
│   │   ├── config-hardened.txt
│   │   ├── config-final.txt
│   │   ├── image-baseline.txt
│   │   ├── image-final.txt
│   │   ├── network-policies-final.txt
│   │   └── nettest-final.txt
│   └── kube-bench/
│       ├── baseline.txt
│       └── baseline-output.log
├── logs/
│   └── remediation-log.md
└── reports/
    ├── final-security-report.md
    ├── validation-evidence.md
    ├── project-scope.md
    └── supporting baseline evidence
```

The original deployment is retained as a baseline for comparison. Apply the
hardened deployment for the final workload; do not apply both versions as
separate running applications because they use the same Deployment name.

## Prerequisites

- Windows 11 with PowerShell
- Docker Desktop
- Minikube
- `kubectl`
- Trivy
- `kube-bench` in a supported Linux/container environment
- Git

The recorded environment used Minikube v1.39.0 and Docker Desktop v29.9.2.
Install and verify current versions before repeating the assessment.

## Quick start

From the repository root in PowerShell:

1. Start Minikube and confirm the active context:

   ```powershell
   minikube start --driver=docker
   kubectl config current-context
   kubectl get nodes
   ```

2. Create the namespace with PSS audit and warning labels:

   ```powershell
   kubectl apply -f manifests/workloads/00-secure-app-namespace.yaml
   ```

   Enforcement is intentionally not enabled by this manifest. Review audit
   output and validate the workload before considering Restricted enforcement.

3. Apply RBAC, network policies, and the hardened application:

   ```powershell
   kubectl apply -f manifests/rbac/
   kubectl apply -f manifests/network-policies/01-default-deny.yaml
   kubectl apply -f manifests/network-policies/02-allow-dns.yaml
   kubectl apply -f manifests/network-policies/03-allow-demo-ingress.yaml
   kubectl apply -f manifests/network-policies/04-allow-nettest-egress.yaml
   kubectl apply -f manifests/workloads/01-demo-hardened.yaml
   kubectl apply -f manifests/workloads/02-demo-service.yaml
   kubectl rollout status deployment/demo-app -n secure-app
   ```

   `00-cluster-export-evidence.yaml` is a historical cluster capture. Do not
   apply it; use the four numbered policy manifests above.

   After server-side validation and review of audit output, Restricted
   enforcement can be enabled explicitly:

   ```powershell
   kubectl label namespace secure-app `
     pod-security.kubernetes.io/enforce=restricted --overwrite
   ```

4. Validate Service selection and endpoints:

   ```powershell
   kubectl -n secure-app get pods -o wide --show-labels
   kubectl -n secure-app get service demo-app -o yaml
   kubectl -n secure-app get endpoints demo-app
   kubectl -n secure-app get networkpolicies
   ```

5. Test the permitted client. The test Pod manifest includes a Restricted-
   compatible security context and the `role=nettest` label required by the
   narrow ingress and egress policies:

   ```powershell
   kubectl apply -f manifests/workloads/03-nettest.yaml
   kubectl -n secure-app exec nettest -- wget -qO- -T 3 http://demo-app:8080
   kubectl -n secure-app label pod nettest role-
   kubectl -n secure-app exec nettest -- wget -qO- -T 3 http://demo-app:8080
   kubectl -n secure-app delete -f manifests/workloads/03-nettest.yaml
   ```

   The first request should succeed. After removing the label, the second
   request should time out or fail because the client no longer matches the
   allow rules.

6. Scan the pinned workload image and manifests. The image scan currently
   reports outstanding high/critical vulnerabilities; `--exit-code 1` makes
   that a failing gate until the image is updated and rescanned:

   ```powershell
   $image = 'docker.io/nginxinc/nginx-unprivileged:mainline-alpine@sha256:b5047b9b76845eac124358373c6852377c52e4b05834a47647e955dd91cb8f1e'
   trivy image --severity HIGH,CRITICAL --exit-code 1 `
     --output scans/trivy/image-final.txt $image
   trivy config --severity MEDIUM,HIGH,CRITICAL --exit-code 1 `
     --output scans/trivy/config-final.txt manifests/workloads/01-demo-hardened.yaml
   trivy config --severity MEDIUM,HIGH,CRITICAL --exit-code 1 `
     --output scans/trivy/network-policies-final.txt manifests/network-policies/
   ```

7. Run kube-bench using the same supported runner and target configuration
   used for the baseline, and keep the raw output:

   ```powershell
   kube-bench > scans/kube-bench/final.txt 2>&1
   ```

   The kube-bench binary generally needs Linux node files and permissions.
   Run it in the project's supported Linux/container environment; do not treat
   a host-side Windows invocation as a valid node audit.

## Milestones and verification status

| Area | Evidence and current status |
|---|---|
| RBAC | A namespace-scoped Role grants `get`, `list`, and `watch` on Pods. The saved `kubectl auth can-i` evidence contains `yes` and `no`; the tested identities/commands were not recorded. |
| Network policies | Default-deny ingress, DNS egress, and narrowly selected app-client rules are organized under `manifests/network-policies/`. The captured cluster output confirms default-deny ingress, but the final policy set has not been re-applied and live-tested in this documentation pass. |
| Image scanning | Baseline `nginx:1.16`: 362 findings (39 critical, 127 high). The digest-pinned unprivileged NGINX candidate scan: 1 critical and 9 high findings. Image remediation is therefore still in progress. |
| Manifest scanning | Original deployment: 18 failures. Current hardened manifest: 1 medium registry-trust finding; NetworkPolicies and Restricted test Pod: 0 findings. Resolve the trusted-registry finding and recheck all final manifests. |
| CIS benchmark | Baseline: 56 pass, 14 fail, and 61 warn. No final kube-bench result is present; CIS remediation is not yet verified. |
| Pod security | The namespace manifest enables Restricted audit and warnings. Enforcement remains a follow-up after server-side validation and warning review. |

See [reports/final-security-report.md](reports/final-security-report.md) for
the project report, [reports/validation-evidence.md](reports/validation-evidence.md)
for recorded and pending tests, and [logs/remediation-log.md](logs/remediation-log.md)
for finding status.
