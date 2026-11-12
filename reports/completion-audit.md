# Completion audit

## Repository status summary

| Requirement area | Status | Evidence | Notes |
|---|---|---|---|
| README project overview and setup | COMPLETE | `README.md` | Updated to describe current implementation and safe validation workflow. |
| RBAC manifest | COMPLETE | `manifests/rbac/rbac.yaml` | Namespace-scoped Role and RoleBinding with least-privilege Pod read access. |
| Network policy manifests | COMPLETE | `manifests/network-policies/*.yaml` | Default deny + DNS + app ingress + nettest egress rules are present. |
| Hardened workload | COMPLETE | `manifests/workloads/01-demo-hardened.yaml` | Non-root, seccomp, no privilege escalation, read-only filesystem, probes, resource limits. |
| Workload service and test pod | COMPLETE | `manifests/workloads/02-demo-service.yaml`, `03-nettest.yaml` | Service and test client are documented and validated. |
| Trivy image scan baseline | COMPLETE | `scans/trivy/image-baseline.txt`, `image-baseline.json` | Original `nginx:1.16` baseline preserved. |
| Trivy hardened manifest scan | COMPLETE | `scans/trivy/config-final.txt` | 1 medium finding remains: KSV-0125 trusted registry warning. |
| Trivy network policy scan | COMPLETE | `scans/trivy/network-policies-final.txt` | Zero findings. |
| kube-bench baseline | PARTIALLY COMPLETE | `scans/kube-bench/baseline.txt`, `baseline-output.log` | Counts are preserved, but the kube-bench release and benchmark profile were not recorded, limiting version-to-version comparison. |
| kube-bench final run | BLOCKED | `scans/kube-bench/final.txt` | Official Aqua image `v0.16.0` works, but its released profiles do not support Kubernetes 1.37; the official node Job also requires unapproved hostPID and sensitive read-only hostPath access. |
| RBAC validation evidence | COMPLETE | `reports/rbac-verification.txt`, new validation notes | Real `kubectl auth can-i` output confirmed least-privilege behavior. |
| Network policy validation evidence | COMPLETE | live cluster test results | Allowed and denied pod-to-pod traffic were both tested and recorded. |
| Environment details | COMPLETE | `reports/environment-details.md` | Actual versions and cluster context captured. |
| Final project reporting | PARTIALLY COMPLETE | existing `reports/final-security-report.md` + generated `.docx` | Final word document is generated; markdown report remains as supporting documentation. |
| Vulnerability tracking spreadsheet | COMPLETE | `reports/vulnerability-remediation.xlsx` | Actual findings captured with evidence references. |
| Validation record | COMPLETE | `reports/validation-evidence.md` | Real cluster outputs captured for the current repo state. |

## Requirement notes

- The project retains a historical baseline for comparison and does not overwrite original evidence.
- The current cluster is a local Minikube environment with Calico as the CNI.
- The app is running in the `secure-app` namespace with restrictive NetworkPolicy rules.
- Full kube-bench remediations are not implemented because they are host-level control-plane changes. The official image was tested, but no supported profile covers the active Kubernetes 1.37 cluster, and the official Job's node access has not been approved.

## Final assessment

The repository has the Kubernetes security manifests, validation evidence, and supporting documentation. The remaining CIS gap is explicit: the current official release does not support the active Kubernetes version, and host-level scanning requires approval for sensitive read-only node access.
