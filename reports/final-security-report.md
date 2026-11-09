# Container Security and Kubernetes Hardening

**Securing a Modern Orchestration Platform**

## Abstract

This project assesses a sample Kubernetes workload and applies controls for
container image risk, RBAC, network traffic, Pod Security Standards, and CIS
Kubernetes Benchmark checks. The saved evidence shows improvement in manifest
configuration and a reduction in image findings after selecting a newer
unprivileged image. The candidate image still has high and critical findings,
and no final cluster or kube-bench run is recorded; those items remain open.

## Introduction

Kubernetes workloads face several related risks:

- Vulnerable container images can expose applications to known software flaws.
- Excessive RBAC permissions can allow compromised users or service accounts
  to read or change resources beyond their task.
- Unrestricted pod traffic can enable lateral movement after a workload is
  compromised.
- Weak pod settings and node configuration can increase the impact of a
  container escape or cluster compromise.

The assessment uses a local, single-node Minikube cluster and a sample NGINX
application in the `secure-app` namespace. Cloud-provider IAM and production
network controls are out of scope.

## Environment Setup & Baseline Audit

The documented toolset is Minikube, Docker Desktop, `kubectl`, Trivy,
`kube-bench`, and Git. The recorded environment lists Minikube v1.39.0 and
Docker Desktop v29.9.2.

The original `nginx:1.16` image scan reported 362 vulnerabilities: 39 critical,
127 high, 149 medium, 38 low, and 9 unknown. The original manifest scan
reported 18 failures across 99 tests. Findings included missing privilege
escalation controls, an unrestricted root filesystem, and no seccomp profile.

The kube-bench baseline recorded 56 pass, 14 fail, and 61 warn results. These
are baseline results only; the repository contains no final kube-bench output.
Some checks may not apply to a local Minikube node and must be assessed against
the benchmark version and cluster configuration.

## Security Controls Implemented

### Container Image Hardening (Trivy fixes, minimal base images)

The hardened Deployment uses the digest-pinned
`docker.io/nginxinc/nginx-unprivileged:mainline-alpine` image rather than the old
`nginx:1.16` baseline. It runs with a non-root UID and has a read-only root
filesystem, dropped capabilities, disabled privilege escalation, resource
requests and limits, and HTTP health probes.

The candidate image scan recorded 1 critical and 9 high vulnerabilities. The
current manifest scan also reports one medium registry-trust finding
(KSV-0125). Image selection and remediation are therefore not complete. Do not
treat digest pinning as evidence that an image is free of vulnerabilities.

### Least-Privilege Access Control (RBAC Roles and Bindings)

The `secure-app` Role permits `get`, `list`, and `watch` on Pods only within
that namespace. A RoleBinding grants those permissions to the
`app-service-account`. The ServiceAccount does not automatically mount its
token. Saved authorization evidence contains `yes` and `no` results, but does
not identify the exact tested subjects or commands.

### Zero-Trust Networking (Default-deny NetworkPolicies, narrow ingress/egress rules)

The network policy set includes default-deny ingress, DNS egress to
`kube-system` on TCP/UDP 53, and narrowly scoped egress/ingress rules for a Pod
labeled `role: nettest` to reach Pods labeled `app: demo` on TCP 8080. The
Service exposes 8080 and targets the workload's named `http` port.

NetworkPolicy behavior depends on a CNI that enforces policies. The manifests
are aligned with the workload labels, but the final policy set has not been
confirmed in a live-cluster test in this documentation pass.

### Pod Security Standards & CIS Remediation

The namespace manifest enables Restricted-profile audit and warnings. It does
not enable enforcement. Validate the workload with server-side dry run, review
warnings, and check cluster compatibility before adding the Restricted
enforcement label.

The kube-bench baseline is retained for remediation planning. There is no final
audit result, so node and control-plane findings are not reported as remediated.

## Results & Validation

### Before-and-after comparison

| Check | Baseline | Current recorded result | Assessment |
|---|---:|---:|---|
| Image vulnerabilities, critical/high | `nginx:1.16`: 39 critical, 127 high | Digest-pinned candidate: 1 critical, 9 high | Improved, not clean; remediation remains open. |
| Trivy configuration scan | Original deployment: 18 failures | Current hardened Deployment: 1 medium KSV-0125; NetworkPolicies and test Pod: 0 findings | UID/GID checks were addressed; resolve or document the trusted-registry finding. |
| kube-bench | 56 pass, 14 fail, 61 warn | No final scan | Not verified. |
| Network policy | Baseline export shows default ingress deny | Final rules are present as manifests; no final live test | Apply and test, including negative traffic tests. |
| RBAC | Saved baseline and `yes`/`no` evidence | Namespace-scoped Pod reader manifest | Re-run `kubectl auth can-i` with documented identities. |
| Pod Security | No recorded enforcement result | Audit/warn labels prepared | Validate first; enforcement is pending. |

### Verified CLI evidence summary

Available evidence files are under `scans/` and `reports/`. The network policy
capture verifies a default ingress-deny policy existed in the cluster at the
time of capture. The RBAC authorization file contains `yes` and `no`; the
identities and resource checks were not included. No post-change network
connectivity test or final kube-bench result is present.

Use [validation-evidence.md](./validation-evidence.md) to record fresh command
output and identify the tested cluster, namespace, and identities.

## Conclusions & Lessons Learned

Security controls must agree on selectors, ports, service accounts, and runtime
settings. Declared container ports do not configure the application listener;
check the Service endpoints and the process listening port when diagnosing
connectivity. Network policies are additive, so an ingress exception does not
grant the client permission to egress.

Image pinning improves reproducibility but does not remediate known CVEs.
Rescan after every image update and keep unresolved findings visible. Treat
baseline and final audits as separate evidence, and mark controls complete
only after they have been tested against the running cluster.
