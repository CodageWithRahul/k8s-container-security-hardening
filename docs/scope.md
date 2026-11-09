# Scope and Risk Context

## 1. Project Purpose
The purpose of this project is to simulate a comprehensive security audit and hardening exercise on a Kubernetes deployment. The objective is to identify security weaknesses in container images, cluster access controls, network communication, and node configurations, and to apply industry-standard security controls following zero-trust and least-privilege principles.

## 2. Project Scope

### In Scope
- Vulnerability scanning and remediation for container images and Kubernetes manifests.
- Role-Based Access Control (RBAC) configurations for users and ServiceAccounts.
- Network Policies governing pod-to-pod and egress/ingress cluster traffic.
- Workload isolation using Kubernetes Pod Security Standards (PSS).
- Control-plane and node configuration auditing against CIS Kubernetes Benchmarks.

### Out of Scope
- Production cloud environments (e.g., AWS EKS, GCP GKE, Azure AKS).
- External network infrastructure, firewalls, or physical hosts beyond the local virtualized environment.
- Cloud Provider IAM integrations.

## 3. Security Objectives
1. **Least Privilege Access:** Enforce granular RBAC policies to restrict permissions for human users and automated workloads.
2. **Zero-Trust Networking:** Eliminate unrestricted pod communication by implementing a default-deny NetworkPolicy and explicit allow rules.
3. **Container Supply Chain Defense:** Eliminate high-severity CVEs in container base images and dependencies.
4. **Control Plane Hardening:** Align cluster components with applicable CIS Kubernetes Benchmark standards.

## 4. Security Assessment Areas

### Container Image Security
Scanning container images and manifests using Trivy to detect known CVEs, outdated packages, and misconfigurations.

### RBAC
Evaluating cluster Roles, ClusterRoles, RoleBindings, and ClusterRoleBindings to eliminate overly permissive privileges (e.g., `*` verbs or cluster-admin bindings).

### Network Security
Auditing and establishing NetworkPolicies to control ingress and egress traffic between namespaces and pods.

### Pod Security
Configuring Pod Security Standards (Privileged, Baseline, Restricted) to limit container capabilities and host namespace exposure.

### Kubernetes Cluster Hardening
Evaluating API server, etcd, controller manager, scheduler, and kubelet configurations using `kube-bench`.

## 5. Tools and Technologies
- **Orchestration:** Minikube (v1.39.0)
- **Container Runtime:** Docker Desktop (v29.9.2)
- **Vulnerability Scanner:** Trivy
- **Compliance Audit Tool:** kube-bench
- **Cluster Administration:** kubectl
- **Version Control:** Git

## 6. Risk Assessment Approach
The risk assessment follows the official project guidelines:
1. Establish Context and Scope
2. Identify Risks and Threats
3. Analyze and Evaluate Risks
4. Determine Risk Treatment
5. Implement Controls and Monitor

## 7. Project Limitations
- Executed on a local Minikube single-node cluster environment.
- Certain CIS Benchmark checks related to multi-node control planes or cloud provider-managed services may be marked as Not Applicable (N/A).