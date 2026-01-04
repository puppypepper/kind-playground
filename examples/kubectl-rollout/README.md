Create a cluster for an example:

```bash
kind create cluster --config kind-config.yaml
```

Check the cluster:

```bash
kubectl cluster-info --context kind-playground
kubectl get nodes
```

Delete the cluster when done:

```bash
kind delete cluster --name playground
```