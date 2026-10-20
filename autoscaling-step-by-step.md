# Kubernetes Autoscaling: Step by Step

Hands-on walkthrough of the Horizontal Pod Autoscaler (HPA) for the `app` Deployment in the `app-demo` namespace, using [`02-autoscaling-availability.yaml`](02-autoscaling-availability.yaml). None of these commands have been run against a cluster by the author, so expect to adjust for your environment.

## How it works

```
metrics-server ──CPU usage──► HPA controller ──sets replicas──► Deployment "app"
   (every ~15s)                 desired = ceil(current × usage / target)
```

With the manifest in this folder:

- Target: **70% average CPU utilization**, measured against the pod's CPU *request*. The app requests `100m`, so the target is about `70m` per pod.
- Range: **2 to 6** replicas.
- Scale up: up to 2 pods per 30 seconds, no delay.
- Scale down: 1 pod per 60 seconds, after a 5 minute stabilization window.

Example: 2 pods at 140% average usage → `ceil(2 × 140 / 70) = 4` replicas.

## Step 1: Prerequisites

The base stack must be running:

```bash
kubectl get pods -n app-demo
```

The HPA computes utilization from the container's CPU **request**. Confirm it is set (the app Deployment sets `cpu: 100m`):

```bash
kubectl get deploy app -n app-demo -o jsonpath='{.spec.template.spec.containers[0].resources.requests}'
```

Without a request the HPA shows `<unknown>` and never scales.

## Step 2: Install metrics-server

The HPA reads pod metrics from the Metrics API. Check whether it already exists:

```bash
kubectl top pods -n app-demo
```

If you see `error: Metrics API not available`, install it:

| Platform | Command |
|----------|---------|
| minikube | `minikube addons enable metrics-server` |
| AKS / GKE / EKS (add-on) | Usually built in; otherwise use the generic install below |
| Generic | `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml` |

On local clusters such as kind or Docker Desktop you may also need the `--kubelet-insecure-tls` argument on the metrics-server container. Wait about a minute, then `kubectl top pods -n app-demo` should print CPU and memory.

## Step 3: Create the HPA

Before applying, one conflict to avoid: the Deployment sets `replicas: 2`. If you re-apply the Deployment later, that value resets the replica count the HPA chose. For a long-lived setup, remove the `replicas` line from [`k8s/app-deployment.yaml`](../k8s/app-deployment.yaml) and let the HPA's `minReplicas` control it.

```bash
kubectl apply -f k8s-scenarios/02-autoscaling-availability.yaml
```

This also creates a PodDisruptionBudget (`minAvailable: 1`), which keeps at least one app pod running during node drains. It is separate from autoscaling.

## Step 4: Inspect it

```bash
kubectl get hpa -n app-demo
```

Expected shape (numbers will differ):

```
NAME      REFERENCE        TARGETS        MINPODS   MAXPODS   REPLICAS
app-hpa   Deployment/app   cpu: 3%/70%    2         6         2
```

If `TARGETS` shows `<unknown>/70%` for more than a minute or two, see Troubleshooting.

```bash
kubectl describe hpa app-hpa -n app-demo      # conditions and scaling events
```

## Step 5: Generate load

Open a second terminal and watch:

```bash
kubectl get hpa app-hpa -n app-demo -w
```

In a third terminal, watch pods:

```bash
kubectl get pods -n app-demo -l app=nodeapp -w
```

`GET /health` is cheap, so use `GET /messages`, which queries PostgreSQL. Start several load generators (works in bash and PowerShell):

```bash
kubectl run load1 -n app-demo --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://app-service/messages >/dev/null; done"
kubectl run load2 -n app-demo --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://app-service/messages >/dev/null; done"
kubectl run load3 -n app-demo --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://app-service/messages >/dev/null; done"
```

If you applied the network policies from scenario 1, these load pods will be blocked by the default-deny egress rule. Delete those policies for this exercise (`kubectl delete -f k8s-scenarios/01-network-policy.yaml`) and re-apply them afterwards.

## Step 6: Observe scale up

Within one to three minutes, `TARGETS` rises above 70% and `REPLICAS` increases, at most 2 pods per 30 seconds, up to 6. Confirm with:

```bash
kubectl top pods -n app-demo
kubectl describe hpa app-hpa -n app-demo     # look for "SuccessfulRescale"
```

If CPU never exceeds 70m per pod, add more load pods (`load4`, `load5`, ...).

If replicas stay below the desired count and some pods are `Pending`, the nodes are out of capacity. That needs the Cluster Autoscaler (see below).

## Step 7: Stop the load and observe scale down

```bash
kubectl delete pod load1 load2 load3 -n app-demo
```

CPU drops immediately, but replicas stay high for about **5 minutes** (the stabilization window), then fall by 1 pod per minute back to 2. This delay is intentional to prevent flapping.

## Step 8: Clean up

```bash
kubectl delete pod load1 load2 load3 -n app-demo --ignore-not-found
kubectl delete -f k8s-scenarios/02-autoscaling-availability.yaml    # removes HPA and PDB
```

## Tuning

| Goal | Change |
|------|--------|
| React faster | Lower `averageUtilization` (for example 50) or raise the scale-up `value` |
| Fewer flaps | Raise `scaleDown.stabilizationWindowSeconds` |
| Higher ceiling | Raise `maxReplicas`; check the ResourceQuota in scenario 4 (`limits.cpu: 6` is about 12 pods at 500m) and PostgreSQL's connection limit (default 100 connections) |
| Scale on memory too | Add a second `Resource` metric with `name: memory`; the HPA uses whichever metric asks for the most replicas |

## The three kinds of autoscaling

| Type | Scales | Use for | Setup |
|------|--------|---------|-------|
| **HPA** | Number of pods | Stateless apps like this Node.js API | This guide |
| **VPA** | Pod CPU/memory requests | Right-sizing; do not combine with HPA on the same CPU/memory metric | VPA add-on |
| **Cluster Autoscaler** | Number of nodes | When scaled-up pods are `Pending` for lack of capacity | Cloud-specific, for example `az aks update --enable-cluster-autoscaler --min-count 1 --max-count 5` |

PostgreSQL in this project is a single replica with a ReadWriteOnce volume, so it should **not** be autoscaled with an HPA. Scale it vertically, or move to a managed database or an operator that supports replicas.

## Troubleshooting

| Symptom | Cause and fix |
|---------|---------------|
| `TARGETS` is `<unknown>/70%` | metrics-server missing or not ready (Step 2), or no CPU `request` on the container |
| `FailedGetResourceMetric` in `describe hpa` | Same as above; check `kubectl get apiservice v1beta1.metrics.k8s.io` shows `True` |
| Never scales up | Load too low; CPU per pod must exceed about 70m. Add load pods |
| Replicas reset to 2 | You re-applied the Deployment with `replicas: 2` (Step 3) |
| Pods stay `Pending` after scale up | Not enough node capacity or the ResourceQuota is exhausted: `kubectl describe pod`, `kubectl describe quota -n app-demo` |
| New pods fail to connect to the DB | PostgreSQL `max_connections` reached; lower `maxReplicas` or add a connection pooler such as PgBouncer |
