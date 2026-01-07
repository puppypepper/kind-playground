Create a cluster for an example:

```bash
kind create cluster --config kind-config.yaml
```

Podを作成する。
```bash
kubectl apply -f manifests/01-app-pod.yaml
```

PodをServiceでexposeする。
```bash
kubectl expose pod nginx-pod --port=80 --name=app-service
```

作成されたServiceを確認する。
`run: nginx-pod`のlabelでPodを識別していることがわかる。
```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2026-01-07T15:16:25Z"
  labels:
    run: nginx-pod
  name: app-service
  namespace: default
  resourceVersion: "1033"
  uid: 2c084c4e-b165-4473-8f6b-a65da50443eb
spec:
  clusterIP: 10.96.139.242
  clusterIPs:
  - 10.96.139.242
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx-pod
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

label無しのPodを作成する。
```bash
kubectl apply -f manifests/02-app-pod-without-label.yaml
```

PodをServiceでexposeする。
labelが存在していないPodはexposeできないということがわかる。
```bash
kubectl expose pod nginx-pod-without-label --port=80 --name=app-without-label-service
```
output
```bash
error: couldn't retrieve selectors via --selector flag or introspection: the pod has no labels and cannot be exposed
```

kind clusterを削除する
```bash
kind delete cluster --name kubectl-expose
```
