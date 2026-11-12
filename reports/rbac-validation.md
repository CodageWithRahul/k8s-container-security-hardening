# RBAC validation

## Test objective

Validate that the `app-service-account` in the `secure-app` namespace has least-privilege read access and cannot perform destructive operations.

## Commands executed

```cmd
kubectl auth can-i get pods --as=system:serviceaccount:secure-app:app-service-account -n secure-app
kubectl auth can-i delete pods --as=system:serviceaccount:secure-app:app-service-account -n secure-app
```

## Actual output

```text
yes
no
```

## Interpretation

- `get pods` is allowed for the `app-service-account` in `secure-app`.
- `delete pods` is denied, which prevents the service account from escalating beyond the intended read-only role.
- The Role in `manifests/rbac/rbac.yaml` is limited to `get`, `list`, and `watch` on Pods in `secure-app`.

## Evidence references

- `manifests/rbac/rbac.yaml`
- `reports/rbac-baseline.txt`
- `reports/rbac-verification.txt`

## Result

PASS: least-privilege RBAC behavior matches the intended access model.
