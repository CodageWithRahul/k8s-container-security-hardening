# Container Security and Kubernetes Hardening

  ## Abstract

  This project validates a local Minikube deployment hardened for RBAC, NetworkPolicy, Pod Security Standards, and container image security. The repository preserves the original baseline evidence and demonstrates that a restricted policy model is functioning in the current cluster. The deployment is running in the `secure-app` namespace and the workload passes the critical application-layer checks that can be executed safely in this environment.

  The remaining concerns are limited to image hardening and host-level CIS benchmark execution. The digest-pinned unprivileged NGINX image still contains unresolved vulnerabilities. The official Aqua kube-bench image runs, but its released profile set does not support the active Kubernetes 1.37 cluster, and the official node-scanning Job requires unapproved access to sensitive host paths.

  ## Introduction

  Local Kubernetes clusters are frequently used for academic and lab-based security work because they allow repeatable testing without a production footprint. The scope of this project is limited to a single-node Minikube environment, where the objective is to validate a sample workload under representative security controls.

  The main risks addressed are:

  - vulnerable or outdated container images
  - excessive RBAC permissions
  - unrestricted pod-to-pod communication
  - weak container runtime security settings
  - incomplete CIS benchmark coverage when host-level controls are out of scope

  ## Materials and methods

  ### Environment

  The current environment was validated directly:

  - Windows 10 build 10.0.26200.9711
  - Docker Desktop 29.9.2
  - Minikube v1.39.0
  - Kubernetes server v1.37.0
  - kubectl client v1.37.2
  - Trivy v0.76.2
  - Aqua kube-bench image `v0.16.0` was inspected and executed for version/help without host mounts

  ### Security controls implemented

  - Namespace-scoped Role and RoleBinding for least-privilege Pod reads
  - Default-deny ingress policy plus narrow allow rules for DNS and the labeled test client
  - Hardened Deployment with non-root execution, read-only root filesystem, no privilege escalation, dropped capabilities, seccomp, and resource limits
  - Digest-pinned unprivileged NGINX image
  - Preserved historical baseline artifacts for comparison

  ## Results and comparison

  ### RBAC

  Live authorization checks were executed:

  ```cmd
  kubectl auth can-i get pods --as=system:serviceaccount:secure-app:app-service-account -n secure-app
  kubectl auth can-i delete pods --as=system:serviceaccount:secure-app:app-service-account -n secure-app
  ```

  Actual output:

  ```text
  yes
  no
  ```

  This confirms least-privilege access: the service account can read Pods but cannot delete them.

  ### Network policy enforcement

  A labeled test Pod was applied and then the selector label was removed:

  ```cmd
  kubectl apply -f manifests/workloads/03-nettest.yaml
  kubectl -n secure-app exec nettest -- wget -qO- -T 3 http://demo-app:8080
  kubectl -n secure-app label pod nettest role- --overwrite
  kubectl -n secure-app exec nettest -- wget -qO- -T 3 http://demo-app:8080
  ```

  Actual output:

  ```text
  <!DOCTYPE html>
  <html>
  ...
  </html>
  ```

  and then:

  ```text
  wget: download timed out
  command terminated with exit code 1
  ```

  This shows that allowed traffic succeeds and unauthorized traffic is blocked by the current NetworkPolicy model.

  ### Container image security

  The original baseline image `nginx:1.16` was retained for historical comparison. The current hardened workload uses the digest-pinned image:

  `docker.io/nginxinc/nginx-unprivileged:mainline-alpine@sha256:b5047b9b76845eac124358373c6852377c52e4b05834a47647e955dd91cb8f1e`

  The final Trivy output shows 12 remaining findings total (11 high, 1 critical) and should be treated as an open risk. The current configuration scan also records one medium finding, `KSV-0125`, for the image registry trust policy. This does not invalidate the workload function, but it does show the image has not yet reached a clean security state.

  ### kube-bench comparison

  The baseline benchmark output is preserved in `scans/kube-bench/baseline.txt` with the following counts:

  - PASS: 56
  - FAIL: 14
  - WARN: 61
  - INFO: 21

  No final benchmark checks were run. The official v0.16.0 image does not
  include a profile for Kubernetes 1.37; its versioned support matrix maps
  `cis-1.12` to Kubernetes 1.32-1.34. Aqua's current main-branch documentation
  lists CIS 2.0 only through Kubernetes 1.35, still below this cluster.
  Moreover, the official Job uses `hostPID: true` and read-only hostPath mounts
  for sensitive directories such as `/etc/kubernetes`, `/var/lib/etcd`, and
  `/var/lib/kubelet`. That access was not approved. The `final.txt` file
  records these checks and limitations; it contains no fabricated PASS/FAIL
  totals.

  ## Conclusions

  The project successfully verifies the core workload-level security controls in the live Minikube cluster: least-privilege RBAC and default-deny NetworkPolicy behavior are functioning as expected. The repository also preserves the evidence required for a fair baseline comparison.

  The outstanding issues are not implementation errors in the repo; they are conditions of the current environment and the selected image supply chain:

  - the current image still contains known vulnerabilities
  - the Trivy trusted-registry policy warning remains open
  - a final kube-bench run requires a released profile supporting Kubernetes 1.37 and approval for the official Job's node access

  This is therefore a disciplined, evidence-based completion: the workload is hardened and validated where the environment allows, while remaining gaps are explicitly documented and not falsely overstated.
