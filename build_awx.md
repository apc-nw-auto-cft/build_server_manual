# building awx

## OS install

rhel系(rocky, alma)の8以上のOSなら多分OK  
CPU 4, Memory 16GB が必要  
[Automation controller system requirements](https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/2.3/html/red_hat_ansible_automation_platform_planning_guide/platform-system-requirements)

rhel系のOS installの注意点として、OS install時のmenuで推奨の作業が複数ある

- NWアダプタをenableする
- userをadmin権限で作成しておく
- timezoneを設定しておく

## 下準備

k3sは軽量なkubernetes

```shell

# k3sのinstall
curl -sfL https://get.k3s.io | sh -

# kustomizeのinstall（tarがないのでそれもinstall）

sudo dnf install tar -y
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash

# kustomizeをpathが通ってるところに移動
sudo mv ./kustomize /usr/bin/

# path通っているか確認
which kustomize

# /usr/bin/kustomizeが出力されればＯＫ


# gitのinstall
dnf install -y git

# firewalld の停止
sudo systemctl disable firewalld --now

```

## kustomization.yamlとawx-demo.yamlを作成

kustomization.yaml

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  - github.com/ansible/awx-operator/config/default?ref=<tag>
  - awx-demo.yaml

# Set the image tags to match the git version from above
images:
  - name: quay.io/ansible/awx-operator
    newTag: <tag>

namespace: awx
```

awx-demo.yaml

```yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx-demo
spec:
  ingress_type: ingress
```

## 立ち上げ

```shell
kubectl apply -k .

# もしpermission errorが出たら
sudo chown $USER:$USER -R /etc/rancher/

# 5分くらいかかる。podの立ち上がりを確認。
# 立ち上がりが完了すると4podがRunningとなる

kubectl get pods -n awx

NAME                                               READY   STATUS    RESTARTS   AGE
awx-operator-controller-manager-6fcb9886d4-mcdnm   2/2     Running   0          6h48m
awx-demo-postgres-13-0                             1/1     Running   0          6h48m
awx-demo-task-58fc65485d-725sq                     4/4     Running   0          6h47m
awx-demo-web-96c7cdd89-gjc74                       3/3     Running   0          6h47m

```

## container registryへのアクセス時に証明書の検証をしない設定

オレオレ証明書環境で、かつCA証明書をAWXのサーバでtrustしてない場合は必要な設定
Dockerhubなどinternet上のcontainer registryを使うなら、CA証明書が信頼済みstoreに存在するため必要ない

/etc/rancher/k3s/registries.yaml を作成し以下を記載

```yaml
---
configs:
  "192.168.79.9:5050":
    tls:
      insecure_skip_vrify: true
```

## passwordの確認

```shell
# passwordは以下。usernameはadmin
kubectl get secret awx-demo-admin-password -o jsonpath="{.data.password}" | base64 --decode ; echo
```
