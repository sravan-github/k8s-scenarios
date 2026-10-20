# Complex Kubernetes Scenarios

Production-style add-ons for the `app-demo` stack (Node.js app + PostgreSQL). Each file is independent; apply only what you need. The base stack in [`k8s/`](../k8s) must be running first. None of these manifests have been applied to a cluster, so test them in a non-production namespace first.

| # | File | Scenario | Needs |
|---|------|----------|-------|
| 1 | [`01-network-policy.yaml`](01-network-policy.yaml) | Zero-trust networking: default deny, then allow only app → postgres, DNS and inbound port 3000 | CNI that enforces NetworkPolicy |
| 2 | [`02-autoscaling-availability.yaml`](02-autoscaling-availability.yaml) | HPA (2–6 replicas on CPU) and PodDisruptionBudget | metrics-server |
| 3 | [`03-scheduling-patch.yaml`](03-scheduling-patch.yaml) | Spread pods across zones/nodes, prefer co-location with the DB | Multi-node cluster |
| 4 | [`04-quota-limits.yaml`](04-quota-limits.yaml) | LimitRange defaults and ResourceQuota | none |
| 5 | [`05-backup-cronjob.yaml`](05-backup-cronjob.yaml) | Nightly `pg_dump` to a PVC with 7-file retention | StorageClass for the PVC |
| 6 | [`06-ingress-tls.yaml`](06-ingress-tls.yaml) | HTTPS Ingress with cert-manager certificates | ingress-nginx, cert-manager, DNS name |
| 7 | [`07-canary-release.yaml`](07-canary-release.yaml) | Canary release via replica ratio | A second image tag |

## Recommended apply order

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f scenarios/04-quota-limits.yaml        # LimitRange before pods are created
kubectl apply -f rbac/rbac.yaml
kubectl apply -f k8s/                                  # base stack
kubectl apply -f scenarios/02-autoscaling-availability.yaml
kubectl patch deployment app -n app-demo --patch-file scenarios/03-scheduling-patch.yaml
kubectl apply -f scenarios/05-backup-cronjob.yaml
kubectl apply -f scenarios/01-network-policy.yaml      # last, so you can debug connectivity first
```

## Scenario notes

### 1. Network policies
`default-deny-all` blocks everything, then targeted policies re-open traffic. The backup job from scenario 5 is already allowed through the `app: pg-backup` label. If you add an ingress controller in another namespace, you must also allow it to reach port 3000 (the current rule allows any source).

Test:
```bash
kubectl run tmp --rm -it --image=busybox:1.36 -n app-demo -- nc -zv postgres-service 5432   # should time out (denied)
kubectl exec -n app-demo deploy/app -- wget -qO- localhost:3000/health                       # still works
```

### 2. Autoscaling and availability
```bash
kubectl get hpa -n app-demo -w
# Generate load, then watch replicas grow
kubectl run load --rm -it --image=busybox:1.36 -n app-demo -- sh -c "while true; do wget -q -O- http://app-service/health; done"
```
Postgres is a single replica, so a PodDisruptionBudget on it would block node drains. It is deliberately omitted.

### 3. Scheduling
Both spread constraints use `ScheduleAnyway`, so pods still schedule on a one-node cluster such as minikube. Check placement with `kubectl get pods -n app-demo -o wide`.

### 4. Quota and limits
The quota counts every container, including init containers. Without the LimitRange defaults the busybox init container (which declares no resources) causes pod creation to fail. Check usage with `kubectl describe quota -n app-demo`.

### 5. Backups
```bash
kubectl create job --from=cronjob/postgres-backup manual-backup -n app-demo   # run now
kubectl logs job/manual-backup -n app-demo
```
Restore into a running database:
```bash
kubectl run restore --rm -it --image=postgres:15-alpine -n app-demo --labels=app=pg-backup \
  --overrides='{"spec":{"volumes":[{"name":"b","persistentVolumeClaim":{"claimName":"postgres-backup-pvc"}}],"containers":[{"name":"restore","image":"postgres:15-alpine","stdin":true,"tty":true,"command":["sh"],"volumeMounts":[{"name":"b","mountPath":"/backup"}]}]}}'
# inside the pod:
#   gunzip -c /backup/<file>.sql.gz | psql -h postgres-service -U postgres appdb
```
A backup on a PVC in the same cluster does not protect against losing the cluster; copy dumps to object storage for real disaster recovery. With `ReadWriteOnce` disks, the restore pod and CronJob should not run at the same time on different nodes.

### 6. Ingress with TLS
Point your DNS `A`/`CNAME` record at the ingress controller's external IP, then watch the certificate: `kubectl get certificate -n app-demo`.

### 7. Canary
```bash
kubectl apply -f scenarios/07-canary-release.yaml
for i in $(seq 1 30); do curl -s http://localhost:8080/health; done    # via port-forward; inspect per-pod logs
kubectl logs -n app-demo -l track=canary
kubectl delete deployment app-canary -n app-demo                       # roll back
```

## Cleanup

```bash
kubectl delete -f scenarios/01-network-policy.yaml -f scenarios/02-autoscaling-availability.yaml \
  -f scenarios/05-backup-cronjob.yaml -f scenarios/07-canary-release.yaml
kubectl delete -f scenarios/04-quota-limits.yaml
kubectl delete -f scenarios/06-ingress-tls.yaml
```
Remove the scheduling patch by re-applying the base `k8s/app-deployment.yaml` with `kubectl replace` or by deleting the `affinity` and `topologySpreadConstraints` fields.
