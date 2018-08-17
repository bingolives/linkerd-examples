# Linkerd1 lifecycle test configuration

Production testing Linkerd1's discovery & caching.

This test is similar to the [Linkerd2](../) lifecycle environment. Differences
are documented here.

## First time setup

[`lifecycle.yml`](lifecycle.yml) creates a `ClusterRole`, which requires your
user to have this ability.

```bash
kubectl create clusterrolebinding cluster-admin-binding-$USER \
  --clusterrole=cluster-admin --user=$(gcloud config get-value account)
```

## Batch Deploy / Scale / Teardown

Deploy a Linkerd daemonset to the `linkerd` namespace:

```bash
kubectl create ns linkerd
kubectl apply -f ../../k8s-daemonset/k8s/linkerd-rbac.yml
kubectl apply -f ../../k8s-daemonset/k8s/servicemesh.yml
```

Deploy the lifecycle environment in 3 namespaces:

```bash
bin/deploy 3
```



Deploy 3 lifecycle environments:

```bash
linkerd install --linkerd-namespace linkerd-lifecycle | kubectl apply -f -
linkerd install --linkerd-namespace linkerd-lifecycle-tls --tls optional | kubectl apply -f -
bin/deploy 3
```

Scale 3 lifecycle environments to 3 replicas of `bb-broadcast`, `bb-p2p`, and
`bb-terminus` each:

```bash
bin/scale 3 3
```

Total mesh-enabled pod count == (1 linkerd ns + 1 linkerd tls ns) * (3*replicas+2)

Teardown 3 lifecycle environments:

```bash
bin/teardown 3

kubectl delete ns linkerd-lifecycle
kubectl delete ns linkerd-lifecycle-tls
```

## Individual Deploy / Scale / Teardown

### Deploy

Install Linkerd service mesh:

```bash
linkerd install --linkerd-namespace linkerd-lifecycle | kubectl apply -f -
linkerd dashboard --linkerd-namespace linkerd-lifecycle
```

Deploy test framework to `lifecycle` namespace:

```bash
export LIFECYCLE_NS=lifecycle
kubectl create ns $LIFECYCLE_NS
cat lifecycle.yml | linkerd inject --linkerd-namespace linkerd-lifecycle - | kubectl -n $LIFECYCLE_NS apply -f -
```

Scale `bb-broadcast`, `bb-p2p`, and `bb-terminus`:

```bash
kubectl -n $LIFECYCLE_NS scale --replicas=3 deploy/bb-p2p deploy/bb-terminus
```

### Observe

Browse to Grafana:

```bash
linkerd dashboard --linkerd-namespace linkerd-lifecycle --show grafana
```

Tail slow-cooker logs:

```bash
kubectl -n $LIFECYCLE_NS logs -f $(
  kubectl -n $LIFECYCLE_NS get po --selector=app=slow-cooker -o jsonpath='{.items[*].metadata.name}'
) slow-cooker
```

Relevant Grafana dashboards to observe
- `Linkerd Deployment`, for route lifecycle and service discovery lifecycle
- `Prometheus 2.0 Stats`, for telemetry resource lifecycle

### Teardown

```bash
kubectl delete ns $LIFECYCLE_NS
kubectl delete ns linkerd-lifecycle
```
