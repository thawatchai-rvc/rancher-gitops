# Graylog on RKE2 + Rancher (Lab / POC only)

Folder layout:

- namespace.yaml — Namespace graylog
- mongodb.yaml — MongoDB Service + StatefulSet + PVC
- opensearch.yaml — OpenSearch Service + StatefulSet + PVC
- configmap.yaml — Graylog non-sensitive settings
- pvc.yaml — Graylog Journal PVC
- deployment.yaml — Graylog Deployment
- service.yaml — Graylog Web NodePort 30900 and Syslog TCP NodePort 31514

## Preparation

1. Confirm default StorageClass: `kubectl get storageclass`
2. On nodes that may schedule OpenSearch, configure persistent `vm.max_map_count=262144`.
3. Change `GRAYLOG_HTTP_EXTERNAL_URI` in configmap.yaml from `NODE_IP` to a reachable node IP or domain.
4. Create admin credentials as a Kubernetes Secret (run in Linux shell with `openssl` and `sha256sum`):

```bash
kubectl apply -f namespace.yaml
read -rsp 'Graylog admin password: ' PASS; echo
HASH=$(printf '%s' "$PASS" | sha256sum | awk '{print $1}')
PEPPER=$(openssl rand -hex 48)
kubectl -n graylog create secret generic graylog-auth \
  --from-literal=GRAYLOG_PASSWORD_SECRET="$PEPPER" \
  --from-literal=GRAYLOG_ROOT_PASSWORD_SHA2="$HASH"
unset PASS HASH PEPPER
```

## Deployment (from this directory)

```bash
kubectl apply -f mongodb.yaml
kubectl apply -f opensearch.yaml
kubectl apply -f configmap.yaml
kubectl apply -f pvc.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get pods,pvc,svc -n graylog
```

Web UI: `http://<NODE-IP>:30900/` (admin; use password you chose).
Syslog endpoint: `<NODE-IP>:31514` TCP. Create a Syslog TCP input on `0.0.0.0:1514` in Graylog UI before sending syslog.

**Lab only:** Single-instance MongoDB and OpenSearch, OpenSearch security disabled. Not production HA, TLS, backups, or a fully hardened deployment. Never commit real secrets to Git. NodePort must not conflict with other Services. App / image behavior and storage permissions may need adjustment for your environment.
