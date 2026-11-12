# Validation evidence

This file records the actual evidence collected for the current Minikube deployment. The results below reflect what was observed in the live cluster and in the saved Trivy output, without claiming unverified remediations.

## 1. Cluster and workload state

- Date/time: 2026-11-12
- Kubernetes context: `minikube`
- Cluster/version: Minikube v1.39.0; Kubernetes server v1.37.0
- Namespace: `secure-app`
- Tested image and digest: `docker.io/nginxinc/nginx-unprivileged:mainline-alpine@sha256:b5047b9b76845eac124358373c6852377c52e4b05834a47647e955dd91cb8f1e`

```text
Command:
kubectl -n secure-app get pods,services,endpoints -o wide

Output:
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME                           READY   STATUS    RESTARTS   AGE   IP              NODE       NOMINATED NODE   READINESS GATES
pod/demo-app-5654694b7-bk2zb   1/1     Running   0          35m   10.244.120.93   minikube   <none>           <none>

NAME               TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE   SELECTOR
service/demo-app   ClusterIP   10.101.77.88   <none>        8080/TCP   35m   app=demo

NAME                 ENDPOINTS            AGE
endpoints/demo-app   10.244.120.93:8080   35m
```

## 2. Pod-to-pod network policy validation

### 2.1 Allowed client request

```text
Command:
kubectl apply -f manifests/workloads/03-nettest.yaml
kubectl -n secure-app exec nettest -- wget -qO- -T 3 http://demo-app:8080

Output:
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</body>
</html>
```

Expected: allowed client succeeds.
Actual: success.
Status: PASS.

### 2.2 Denied client request after selector label removal

```text
Command:
kubectl -n secure-app label pod nettest role- --overwrite
kubectl -n secure-app exec nettest -- wget -qO- -T 3 http://demo-app:8080

Output:
wget: download timed out
command terminated with exit code 1
```

Expected: label mismatch stops traffic.
Actual: timeout.
Status: PASS.

## 3. RBAC validation

```text
Command:
kubectl auth can-i get pods --as=system:serviceaccount:secure-app:app-service-account -n secure-app
kubectl auth can-i delete pods --as=system:serviceaccount:secure-app:app-service-account -n secure-app

Output:
yes
no
```

Expected: `yes` for `get pods`, `no` for `delete pods`.
Actual: match expected result.
Status: PASS.

## 4. Trivy scan summary

### 4.1 Final image scan

```text
Target: docker.io/nginxinc/nginx-unprivileged:mainline-alpine@sha256:b5047b9b76845eac124358373c6852377c52e4b05834a47647e955dd91cb8f1e
Total: 12 (HIGH: 11, CRITICAL: 1)
Report: scans/trivy/image-final.txt
```

This confirms the image is still not vulnerability-free. The finding remains open and should be tracked until a safer image version is adopted or a vendor-supplied patch removes the exposed packages.

### 4.2 Final config scan

```text
Target: manifests/workloads/01-demo-hardened.yaml
Failures: 1 (MEDIUM: 1, HIGH: 0, CRITICAL: 0)
Finding: KSV-0125 - image from untrusted registry
Report: scans/trivy/config-final.txt
```

The remaining medium finding is the trusted-registry policy check for the image source. It is a configuration policy finding rather than an app runtime failure.

### 4.3 Final network policy scan

```text
Target: manifests/network-policies/
Misconfigurations: 0
Report: scans/trivy/network-policies-final.txt
```

Status: PASS.

## 5. kube-bench baseline and final status

Baseline summary from `scans/kube-bench/baseline.txt`:

- PASS: 56
- FAIL: 14
- WARN: 61
- INFO: 21

The official Aqua image was available and its CLI was tested without host
access:

```text
Command:
docker run --rm docker.io/aquasec/kube-bench:v0.16.0 version

Output:
v0.16.0

Image digest:
sha256:75506f222d1eb6ce2a751a5533bdc0a3b54c898e2e49e7751d0ee22cfb862679
```

No CIS check was run. The image does not include a Kubernetes 1.37 profile;
its v0.16.0 support table maps its newest shipped CIS profile to Kubernetes
1.32-1.34. The official Job requires `hostPID: true` and read-only mounts of
sensitive node directories, which were not approved. Do not interpret the
absence of a final run as zero findings. Exact evidence is in
`scans/kube-bench/final.txt`.

## 6. Result overview

| Check | Status | Evidence |
|---|---|---|
| Workload availability | PASS | `kubectl -n secure-app get pods,services,endpoints -o wide` |
| Allowed pod-to-pod traffic | PASS | `wget` returned NGINX HTML |
| Denied pod-to-pod traffic | PASS | `wget` timed out after label removal |
| RBAC least privilege | PASS | `kubectl auth can-i` output `yes` / `no` |
| Image vulnerability reduction | PARTIALLY COMPLETE | final scan still shows 12 findings |
| Manifest policy compliance | PASS | Zero findings in network-policy scan |
| Final kube-bench run | BLOCKED | released profile does not support Kubernetes 1.37; sensitive node access requires approval |
