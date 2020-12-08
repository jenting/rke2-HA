# rke2-HA

The control plane nodes _must be_ odd number.
```shell
# 1st control plane node
curl -sfL https://get.rke2.io | sh -
systemctl enable rke2-server.service
systemctl start rke2-server.service

# 2nd, 3rd control plane nodes
curl -sfL https://get.rke2.io | sh -
systemctl enable rke2-server.service

mkdir -p /etc/rancher/rke2/
vim /etc/rancher/rke2/config.yaml
server: https://<server>:9345
token: <token from server node> # cat /etc/rancher/rke2/config.yaml

systemctl start rke2-server.service

# worker nodes
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
systemctl enable rke2-agent.service

mkdir -p /etc/rancher/rke2/
vim /etc/rancher/rke2/config.yaml
server: https://<server>:9345
token: <token from server node> # cat /etc/rancher/rke2/config.yaml

systemctl start rke2-agent.service
```

Install the system-upgrade-controller.
```shell
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/download/v0.6.2/system-upgrade-controller.yaml

cat <<EOF | kubectl apply -f -
# Server plan
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: server-plan
  namespace: system-upgrade
spec:
  concurrency: 1
  cordon: true
  nodeSelector:
    matchExpressions:
    - key: node-role.kubernetes.io/master
      operator: In
      values:
      - "true"
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/rke2-upgrade
  channel: https://update.rke2.io/v1-release/channels/stable
---
# Agent plan
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: agent-plan
  namespace: system-upgrade
spec:
  concurrency: 1
  cordon: true
  nodeSelector:
    matchExpressions:
    - key: node-role.kubernetes.io/master
      operator: DoesNotExist
  prepare:
    args:
    - prepare
    - server-plan
    image: rancher/rke2-upgrade:v1.17.4-k3s1
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/rke2-upgrade
  channel: https://update.rke2.io/v1-release/channels/stable
EOF
```
