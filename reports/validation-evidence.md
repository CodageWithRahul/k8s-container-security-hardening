# Validation evidence

Use this log to record repeatable test commands, expected behavior, actual
output, date, cluster context, and namespace. Replace each placeholder with
fresh output; do not present expected output as a completed test.

## Cluster and workload

- Date/time: `[enter date and time]`
- Kubernetes context: `[kubectl config current-context]`
- Cluster/version: `[enter Minikube and Kubernetes versions]`
- Namespace: `secure-app`
- Tested image and digest: `[enter image reference]`

```text
Command:
kubectl -n secure-app get pods,services,endpoints -o wide

Output:
[paste output]
```

## Pod-to-pod network policy tests

Allowed test using the labeled client manifest:

```powershell
kubectl apply -f manifests/workloads/03-nettest.yaml
kubectl -n secure-app exec nettest -- wget -qO- -T 3 http://demo-app:8080
```

Expected: `wget` returns the application response and exits successfully.

```text
Date/time:
Context:
Command:
kubectl -n secure-app exec nettest -- wget -qO- -T 3 http://demo-app:8080

Output:
[paste actual output]
Result: [PASS / FAIL]
```

Denied test: remove the selector label from the running test Pod and repeat the
same request. The client should no longer match either app allow rule.

```text
Date/time:
Context:
Commands:
kubectl -n secure-app label pod nettest role-
kubectl -n secure-app exec nettest -- wget -qO- -T 3 http://demo-app:8080

Output:
[paste actual timeout/refusal and Pod events]
Result: [PASS / FAIL]
```

## RBAC validation

Use the intended service account and a resource/verb pair in the target
namespace:

```powershell
kubectl auth can-i get pods `
  --as=system:serviceaccount:secure-app:app-service-account `
  -n secure-app
kubectl auth can-i delete pods `
  --as=system:serviceaccount:secure-app:app-service-account `
  -n secure-app
```

Expected: `yes` for `get pods`; `no` for `delete pods`.

```text
Date/time:
Context:
Commands:
[paste commands]

Output:
[paste actual output]
Result: [PASS / FAIL]
```

The historical `reports/rbac-verification.txt` contains `yes` and `no`, but
does not identify the subjects or checks that produced those results.

## Pod Security Standards

```powershell
kubectl apply --dry-run=server -f manifests/workloads/01-demo-hardened.yaml
kubectl get namespace secure-app --show-labels
```

```text
Date/time:
Context:
Commands:
[paste commands]

Output:
[paste actual output, including PSS warnings]
Result: [PASS / FAIL]
Restricted enforcement enabled: [yes / no]
```

## Trivy image and configuration scans

### Results recorded during repository preparation

| Scan | Result | Output |
|---|---|---|
| Baseline image `nginx:1.16` | 362 total: 39 critical, 127 high, 149 medium, 38 low, 9 unknown | `scans/trivy/image-baseline.txt` |
| Pinned unprivileged NGINX image | 57 total: 1 critical, 9 high, 3 medium, 44 low | `scans/trivy/image-final.txt` and `image-final.json` |
| Hardened Deployment configuration | 98 tests: 97 successes, 1 medium failure (KSV-0125, registry trust) | `scans/trivy/config-final.txt` |
| NetworkPolicy configuration | 0 misconfigurations across four policies | `scans/trivy/network-policies-final.txt` |
| Restricted test Pod configuration | 0 misconfigurations | `scans/trivy/nettest-final.txt` |

The image and registry-trust findings remain unresolved. These are local scan
results only; they do not prove that the manifests have been applied or tested
on the cluster.

```text
Image reference and digest:
[enter exact image]

Command:
[paste command]

Summary:
[paste total and severity counts]

Report path:
[enter path under scans/trivy/]
```

```text
Manifest scan command:
[paste command]

Summary:
[paste passed/failed checks and findings]

Report path:
[enter path under scans/trivy/]
```

## kube-bench baseline vs. final

| Result | PASS | FAIL | WARN | INFO | Report |
|---|---:|---:|---:|---:|---|
| Baseline | 56 | 14 | 61 | 0 | `scans/kube-bench/baseline.txt` |
| Final | `[enter]` | `[enter]` | `[enter]` | `[enter]` | `scans/kube-bench/final.txt` |

Compare only checks applicable to the same benchmark version and cluster
configuration. Record N/A checks and remediation evidence:

```text
Benchmark/version:
Cluster:
Command:
[paste command]

Summary/diff:
[paste actual output]

Applicable failures remaining:
[list checks or state none]
```

## Historical evidence notes

- `reports/network-policy-baseline.txt` records the default ingress-deny policy
  from the cluster before the final policy set was documented.
- `reports/rbac-baseline.txt` and `reports/rbac-verification.txt` preserve the
  prior RBAC inspection and short authorization results.
- `manifests/network-policies/00-cluster-export-evidence.yaml` is a captured
  cluster export, retained for evidence only. Do not apply it; use the four
  numbered policy manifests for new deployments.
- `scans/trivy/` and `scans/kube-bench/` contain the retained baseline scan
  output. No final kube-bench output is currently recorded.
