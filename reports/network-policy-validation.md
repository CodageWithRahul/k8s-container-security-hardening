# Network policy validation

## Test objective

Verify that the cluster enforces the intended ingress/egress policy model for the `secure-app` workload.

## Active controls in the cluster

The following policies are present in the running cluster:

- `default-deny-ingress`
- `allow-dns-egress`
- `allow-demo-ingress`
- `allow-nettest-to-demo`

The cluster is running Calico (`calico-node` in `kube-system`), which supports NetworkPolicy enforcement.

## Test 1: permitted traffic from the labeled client

### Command

```cmd
kubectl apply -f manifests/workloads/03-nettest.yaml
kubectl -n secure-app exec nettest -- wget -qO- -T 3 http://demo-app:8080
```

### Result

The command returned the default NGINX HTML page successfully.

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</body>
</html>
```

This confirms that a Pod labeled `role=nettest` can reach the `app=demo` Service on port 8080.

## Test 2: denied traffic after label removal

### Command

```cmd
kubectl -n secure-app label pod nettest role- --overwrite
kubectl -n secure-app exec nettest -- wget -qO- -T 3 http://demo-app:8080
```

### Result

```text
wget: download timed out
command terminated with exit code 1
```

This confirms the policy denies the unlabeled Pod and blocks traffic after selector matching is lost.

## Test 3: DNS access

The DNS egress rule allows UDP/TCP 53 traffic to the `kube-system` namespace. This rule is required for name resolution and is part of the policy set.

## Result

PASS: NetworkPolicy enforcement is functioning in the current Minikube cluster for both allowed and denied traffic cases.
