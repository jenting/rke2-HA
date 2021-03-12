# rke2-HA

The control plane nodes _must be_ odd number.
```shell
# 1st control plane node
sudo su -

systemctl disable firewalld
systemctl stop firewalld

curl -sfL https://get.rke2.io | sh -

systemctl enable rke2-server.service
systemctl start rke2-server.service

SERVER_NODE_TOKEN=`cat /var/lib/rancher/k3s/server/node-token`

# 2nd, 3rd control plane nodes
sudo su -

systemctl disable firewalld
systemctl stop firewalld

curl -sfL https://get.rke2.io | sh -

systemctl enable rke2-server.service

mkdir -p /etc/rancher/rke2/
vim /etc/rancher/rke2/config.yaml
server: https://<LB_IP>:9345
token: $SERVER_NODE_TOKEN

systemctl start rke2-server.service

# worker nodes
systemctl disable firewalld
systemctl stop firewalld
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
systemctl enable rke2-agent.service

mkdir -p /etc/rancher/rke2/
cat << EOF > /etc/rancher/rke2/config.yaml
server: https://<LB_IP>:9345
token: $SERVER_NODE_TOKEN
EOF

systemctl start rke2-agent.service
```

# rke2-HA with CIS-1.5 profile

The control plane nodes _must be_ odd number.
```shell
# 1st control plane node
sudo su -

systemctl disable firewalld
systemctl stop firewalld

curl -sfL https://get.rke2.io | sh -

systemctl enable rke2-server.service

cat << EOF > /etc/sysctl.d/60-rke2-cis.conf
vm.panic_on_oom=0
vm.overcommit_memory=1
kernel.panic=10
kernel.panic_on_oops=1
EOF
sudo systemctl restart systemd-sysctl

useradd -r -c "etcd user" -s /sbin/nologin -M etcd

mkdir -p /etc/rancher/rke2/
cat << EOF > /etc/rancher/rke2/config.yaml
profile: "cis-1.5"
EOF

systemctl start rke2-server.service

SERVER_NODE_TOKEN=`cat /var/lib/rancher/k3s/server/node-token`

# 2nd, 3rd control plane nodes
sudo su -

systemctl disable firewalld
systemctl stop firewalld

curl -sfL https://get.rke2.io | sh -

systemctl enable rke2-server.service

cat << EOF > /etc/sysctl.d/60-rke2-cis.conf
vm.panic_on_oom=0
vm.overcommit_memory=1
kernel.panic=10
kernel.panic_on_oops=1
EOF
sudo systemctl restart systemd-sysctl

useradd -r -c "etcd user" -s /sbin/nologin -M etcd

mkdir -p /etc/rancher/rke2/
cat << EOF > /etc/rancher/rke2/config.yaml
server: https://<LB_IP>:9345
token: $SERVER_NODE_TOKEN
profile: "cis-1.5"
EOF

systemctl start rke2-server.service

# worker nodes
sudo su -

systemctl disable firewalld
systemctl stop firewalld

curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -

systemctl enable rke2-agent.service

cat << EOF > /etc/sysctl.d/60-rke2-cis.conf
vm.panic_on_oom=0
vm.overcommit_memory=1
kernel.panic=10
kernel.panic_on_oops=1
EOF
sudo systemctl restart systemd-sysctl

mkdir -p /etc/rancher/rke2/
cat << EOF > /etc/rancher/rke2/config.yaml
server: https://<LB_IP>:9345
token: $SERVER_NODE_TOKEN
profile: "cis-1.5"
EOF

systemctl start rke2-agent.service
```

## Install system-upgrade-controller
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
  concurrency: 2
  cordon: true
  drain:
    force: true
  nodeSelector:
    matchExpressions:
    - key: node-role.kubernetes.io/master
      operator: NotIn
      values:
      - "true"
  prepare:
    args:
    - prepare
    - server-plan
    image: rancher/rke2-upgrade
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/rke2-upgrade
  channel: https://update.rke2.io/v1-release/channels/stable
EOF
```

References:
1. https://docs.rke2.io/
2. https://github.com/rancher/system-upgrade-controller
3. https://github.com/rancher/rke2-upgrade
