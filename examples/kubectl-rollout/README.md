Create a cluster for an example:

```bash
kind create cluster --config kind-config.yaml
```

Deploymentを作成する。
```bash
kubectl apply -f manifests/01-app-deployment.yaml
```

DeploymentによってPodが作成されていることを確認する。
```bash
kubectl get pods
```

output
```bash
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-54d64cbd6f-5r8s7   1/1     Running   0          2m57s
app-deployment-54d64cbd6f-6wwrc   1/1     Running   0          2m57s
app-deployment-54d64cbd6f-drtj5   1/1     Running   0          2m57s
```

Deploymentのrollout履歴を確認する
```bash
kubectl rollout history deployment app-deployment
```
output
```bash
deployment.apps/app-deployment 
REVISION  CHANGE-CAUSE
1         <none>
```

```bash
kubectl set image deployment/app-deployment nginx=nginx:1.19
```
output
```bash
deployment.apps/app-deployment image updated
```

この時点でDeploymentのimageは更新されていることを確認する
```bash
kubectl get deployment -o wide
```
output
```bash
NAME             READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES       SELECTOR
app-deployment   3/3     3            3           113s   nginx        nginx:1.19   app=app-deployment
```

rollout historyにもREVISIONが追加されている
```bash
kubectl rollout history deployment app-deployment
```
output
```bash
deployment.apps/app-deployment 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

rollout historyの詳細を確認することもできる
```bash
kubectl rollout history deployment app-deployment --revision 2
```
output
```bash
deployment.apps/app-deployment with revision #2
Pod Template:
  Labels:       app=app-deployment
        pod-template-hash=5f7d6bbff7
  Containers:
   nginx:
    Image:      nginx:1.19
    Port:       <none>
    Host Port:  <none>
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
  Node-Selectors:       <none>
  Tolerations:  <none>
```

kind clusterを削除する
```bash
kind delete cluster --name kubectl-rollout
```