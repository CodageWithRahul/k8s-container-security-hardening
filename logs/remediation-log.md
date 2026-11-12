# Remediation log

This log records the actual state of the project after live validation.

| Source | Target / Image | Finding | Severity | Remediation performed | Status |
|---|---|---|---|---|---|
| Trivy image baseline | `nginx:1.16` | 362 total findings | Critical / High | Baseline preserved; original image remains historical. | Baseline retained |
| Hardened image candidate | `docker.io/nginxinc/nginx-unprivileged:mainline-alpine@sha256:b5047b9b76845eac124358373c6852377c52e4b05834a47647e955dd91cb8f1e` | 12 total findings (1 critical, 11 high) | Critical / High | Digest-pinned image applied to the workload. | Open; still not vulnerability-free |
| Hardened workload config | `01-demo-hardened.yaml` | KSV-0125 trusted-registry finding | Medium | Secured runtime settings were applied; registry trust is still a policy concern. | Open, but not a runtime failure |
| RBAC | `secure-app` Role + RoleBinding | `get` allowed, `delete` denied | Low / Medium | Namespace-scoped read-only Role implemented | Verified in cluster |
| Network policy | `secure-app` policies | Allowed nettest request; denied unlabeled request | High | Default deny + DNS + app allow rules implemented | Verified in cluster |
| kube-bench | Minikube host | baseline capture exists; final run blocked | High / Medium | Baseline preserved; no unsafe host changes made | Blocked by missing kube-bench binary |
| Pod Security Standards | `secure-app` namespace | Audit/warn labels configured | Medium | Namespace labels established for PSS review | Validation pending full Restricted enforcement |

## Current project assessment

The repository is functionally hardened at the workload and namespace level, and the live cluster validation confirms the intended behavior for RBAC and NetworkPolicy controls. The work that remains open is limited to:

- replacing or upgrading the current unprivileged image until the remaining CVEs are resolved
- reviewing the trusted-registry policy warning from Trivy
- running kube-bench in a Linux-capable environment to complete host-level CIS evaluation
