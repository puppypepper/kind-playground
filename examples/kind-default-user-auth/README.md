# 前提条件

CLIで以下を使うことができるようにしてください。
- `openssl`

# kindデフォルトユーザー認証認可確認シナリオ

kindで今回使用するclusterを作成する。
```bash
kind create cluster --config kind-config.yaml
```

kindでcluster作成直後のuser, contextを確認する。
```bash
kubectl config view
```
output
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://127.0.0.1:52892
  name: kind-kind-default-user-auth
contexts:
- context:
    cluster: kind-kind-default-user-auth
    user: kind-kind-default-user-auth
  name: kind-kind-default-user-auth
current-context: kind-kind-default-user-auth
kind: Config
preferences: {}
users:
- name: kind-kind-default-user-auth
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
```

`kind-kind-default-user-auth`というユーザー名で、クライアント証明書を使って認証を行っているという事がわかる。
使用しているクライアント証明書の詳細を確認する。

```bash
kubectl config view --raw | \
  yq -r '.users[] | select(.name == "kind-kind-default-user-auth") | .user.client-certificate-data' | \
  base64 --decode | \
  openssl x509 -noout -text
```
output 
重要な部分を抜粋
```text
...
        Issuer: CN=kubernetes
        Validity
            Not Before: Jan 17 09:05:14 2026 GMT
            Not After : Jan 17 09:10:14 2027 GMT
        Subject: O=kubeadm:cluster-admins, CN=kubernetes-admin
...
```

Username: `kubernetes-admin`、Group: `kubeadm:cluster-admins`として認証されるクライアント証明書であることを示している。
kubectlで実際のUsernameとGroupを確認する。

```bash
kubectl auth whoami
```
output
```text
ATTRIBUTE                                           VALUE
Username                                            kubernetes-admin
Groups                                              [kubeadm:cluster-admins system:authenticated]
Extra: authentication.kubernetes.io/credential-id   [X509SHA256=463da8c2a00f8cc9408c03bbb5b945b5048f37e9fbdeee901e1d74f4f9f1a364]
```

クライアント証明書によって認証されたことにより`system:authenticated`というGroupが追加で付与されているが、それ以外はクライアント証明書で確認した内容と一致している。

次に、デフォルトで作成されたユーザの権限を確認する。
```bash
kubectl auth can-i --list
```
output
```text
Resources                                       Non-Resource URLs   Resource Names   Verbs
*.*                                             []                  []               [*]
                                                [*]                 []               [*]
selfsubjectreviews.authentication.k8s.io        []                  []               [create]
selfsubjectaccessreviews.authorization.k8s.io   []                  []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                  []               [create]
                                                [/api/*]            []               [get]
                                                [/api]              []               [get]
                                                [/apis/*]           []               [get]
                                                [/apis]             []               [get]
                                                [/healthz]          []               [get]
                                                [/healthz]          []               [get]
                                                [/livez]            []               [get]
                                                [/livez]            []               [get]
                                                [/openapi/*]        []               [get]
                                                [/openapi]          []               [get]
                                                [/readyz]           []               [get]
                                                [/readyz]           []               [get]
                                                [/version/]         []               [get]
                                                [/version/]         []               [get]
                                                [/version]          []               [get]
                                                [/version]          []               [get]
```

このように強い権限を与えられていることがわかる。
この権限を現在のユーザーに紐づけているClusterRoleBindingリソースを確認する。

```bash
kubectl get clusterrolebindings -o yaml | \
  yq '.items[]
  | select(
      .subjects[]?.name == "kubernetes-admin"
      or .subjects[]?.name == "kubeadm:cluster-admins"
    )
  | .metadata.name'
```
output
```text
kubeadm:cluster-admins
```

```bash
kubectl get clusterrolebindings kubeadm:cluster-admins -o yaml 
```
output
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  creationTimestamp: "2026-01-17T09:48:19Z"
  name: kubeadm:cluster-admins
  resourceVersion: "207"
  uid: a5115572-c406-4a67-b05f-fd15f0337542
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: kubeadm:cluster-admins
```

ClusterRoleBindingの詳細確認。このClusterRoleBindingによって基本的なリソースの操作権限が付与されていることがわかる。
```bash
kubectl get clusterroles cluster-admin -o yaml
```
output
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2026-01-17T09:48:18Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: cluster-admin
  resourceVersion: "74"
  uid: 6a254fae-dc23-4820-bfee-21d8548e2ebf
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs:
  - '*'
  verbs:
  - '*'
```

system:authenticatedによって付与されているClusterRoleBindingも同様にして確認することができる。
```bash
kubectl get clusterrolebindings -o yaml | \
  yq '.items[]
  | select(
      .subjects[]?.name == "system:authenticated"
    )
  | .metadata.name'
```
output
```text
system:basic-user
system:discovery
system:public-info-viewer
```

*以降省略。

kind clusterを削除する
```bash
kind delete cluster --name kind-default-user-auth
```