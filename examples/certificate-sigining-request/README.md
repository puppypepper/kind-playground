# 前提条件

CLIで以下を使うことができるようにしてください。
- `yq`
- `openssl`

事前に`kind-default-user-auth`のシナリオを確認することを推奨します。

# CertificateSigningRequest確認シナリオ

kindで今回使用するclusterを作成する。
```bash
kind create cluster --config kind-config.yaml
```

CSRに使用する秘密鍵を作成する。
```bash
openssl genrsa -out resources/csr-user.key 2048
```

作成した秘密鍵を用いてCSRを作成する。
Common Name: `csr-user`
Organization: `csr-group`
```bash
openssl req -new -key resources/csr-user.key -out resources/csr-user.csr \
  -subj "/CN=csr-user/O=csr-group"
```

CSRをKubernetesリソースに合わせて整形する。
```bash
base64 < resources/csr-user.csr | tr -d '\n' > resources/csr-user.csr.b64
```

kind: CertificateSigningRequestのmanifestに値を適用する。
```bash
yq -i '.spec.request = load_str("resources/csr-user.csr.b64")' manifests/csr.yaml
```

Kubernetesクラスタにmanifestを適用する。
```bash
kubectl apply -f manifests/csr.yaml
```
output
```text
certificatesigningrequest.certificates.k8s.io/csr-user created
```

kubectlでリソースを確認する。
```bash
kubectl get csr csr-user
```
output
```text
NAME       AGE   SIGNERNAME                            REQUESTOR          REQUESTEDDURATION   CONDITION
csr-user   76s   kubernetes.io/kube-apiserver-client   kubernetes-admin   <none>              Pending
```

この状態ではPendingとなっているためクライアント証明書を利用してkube-apiserverにアクセスすることはできない。
kindのdefaultユーザーとしてCSRのapproveを行う。
```bash
kubectl certificate approve csr-user
```
output
```text
certificatesigningrequest.certificates.k8s.io/csr-user approved
```

リソースの状態がPendingからApproved,Issuedに変わっていることを確認する。
```bash
kubectl get csr csr-user
```
output
```text
NAME       AGE     SIGNERNAME                            REQUESTOR          REQUESTEDDURATION   CONDITION
csr-user   4m15s   kubernetes.io/kube-apiserver-client   kubernetes-admin   <none>              Approved,Issued
```

リソースの詳細を確認し、status.certificateにクライアント証明書が作成されていることを確認する。
```bash
kubectl get csr csr-user -o yaml
```
output
```text
...
status:
  certificate: <Your Base64 encoded cert> 
```

作成されたクライアント証明書をファイルとして保存する。
```bash
kubectl get csr csr-user \
  -o jsonpath='{.status.certificate}' \
  | base64 --decode > resources/csr-user.crt
```

クライアント証明書の確認を行う。CSRを作成した際に設定したSubjectに対して証明書が発行されていることが確認できる。
```bash
openssl x509 -in resources/csr-user.crt -noout -text
```
output
重要な部分を抜粋
```text
...
        Issuer: CN=kubernetes
        Validity
            Not Before: Jan 17 10:18:06 2026 GMT
            Not After : Jan 17 10:18:06 2027 GMT
        Subject: O=csr-group, CN=csr-user
...
```

作成したユーザーをkubeconfigに登録する。
```bash
kubectl config set-credentials csr-user \
  --client-certificate=resources/csr-user.crt \
  --client-key=resources/csr-user.key
```
output
```text
User "csr-user" set.
```

作成したcsr-userとしてkubectlコマンドを利用する。
```bash
kubectl --user=csr-user auth whoami
```
output
```text
ATTRIBUTE                                           VALUE
Username                                            csr-user
Groups                                              [csr-group system:authenticated]
Extra: authentication.kubernetes.io/credential-id   [X509SHA256=997c816e61503713c31e50575ab94aac542ccf668db57bf819076ce841c90a1b]
```

作成したユーザーは特にClusterRoleBindingなどで権限を紐づけていないためPod一覧の取得なども実行することができない。
認可の適用に関しては別のシナリオで対応する。
```bash
kubectl --user=csr-user get pods         
```
output
```text
Error from server (Forbidden): pods is forbidden: User "csr-user" cannot list resource "pods" in API group "" in the namespace "default"
```

kube-configファイルからcsr-userを削除する。
```bash
kubectl config unset users.csr-user
```
output
```text
Property "users.csr-user" unset.
```

kind clusterを削除する
```bash
kind delete cluster --name certificate-signing-request
```