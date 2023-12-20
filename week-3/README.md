# day 3

## backup and restore

## certificate rotation

## log management

## upgrading cluster

https://github.com/rancher/system-upgrade-controller


## advanced configuration

--node-tain during install # mark node as don't put workload on node


customize `containerd` -> e.g. use differen registry


`local path provisioner` -> default storage class
provision local volumes on node-filsystem

`hostPath`?!

=> Longhorn -> distributed block storaage, block-level replication, designed for k8s



## challenge 8: backup and restore k3s SQLITE

```sh
ssh-keygen -R 192.168.56.11 \
&& ssh-keygen -R 192.168.56.12

ssh-copy-id -i ~/.ssh/id_ed25519.pub root@192.168.56.11 \
  && ssh-copy-id -i ~/.ssh/id_ed25519.pub root@192.168.56.12

k3sup install \
  --ssh-key ~/.ssh/id_ed25519 \
  --ip=192.168.56.11 \
  --user=root \
  --k3s-channel=stable \
  --local-path=k3s-server-1.yaml \
  --context k3s-server-1 \
  --k3s-extra-args="--node-external-ip 192.168.56.11 --node-ip 192.168.56.11"

k3sup join \
  --ssh-key ~/.ssh/id_ed25519 \
  --ip=192.168.56.12 \
  --server-user=root \
  --server-ip=192.168.56.11 \
  --user=root \
  --k3s-channel=stable \
  --k3s-extra-args="--node-external-ip 192.168.56.12 --node-ip 192.168.56.12"

export KUBECONFIG=$(pwd)/k3s-server-1.yaml
```
## challenge 8: backup and restore k3s ETCD

```sh
k3sup install \
  --ssh-key ~/.ssh/id_ed25519 \
  --ip=192.168.56.11 \
  --user=root \
  --k3s-channel=stable \
  --cluster `# init etcd-backend` \
  --local-path=k3s-server-1.yaml \
  --context k3s-server-1 \
  --k3s-extra-args="--node-external-ip 192.168.56.11 --node-ip 192.168.56.11"

k3sup join \
  --ssh-key ~/.ssh/id_ed25519 \
  --ip=192.168.56.12 \
  --server-user=root \
  --server-ip=192.168.56.11 \
  --user=root \
  --k3s-channel=stable \
  --server `# join etcd via server`\
  --k3s-extra-args="--node-external-ip 192.168.56.12 --node-ip 192.168.56.12"


vi  /etc/systemd/system/k3s.service

  '--etcd-snapshot-schedule-cron' \
  '*/2 * * * *' \

systemctl daemon-reload && systemctl restart k3s

ls /var/lib/rancher/k3s/server/db/snapshots

kubectl create deploy busybox --image=busybox -- sleep 3600

kubectl delete deploy busybox

k3s server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/etcd-snapshot-k3s-server-1-1643738640 \
  --node-external-ip 192.168.56.11 \
  --node-ip 192.168.56.11

systemctl start k3s

```

## Backing Up the Data Store May Not Be Enough - simulating node failur

```sh
ssh-keygen -R 192.168.56.11 \
&& ssh-keygen -R 192.168.56.12

ssh-copy-id -i ~/.ssh/id_ed25519.pub root@192.168.56.11 \
  && ssh-copy-id -i ~/.ssh/id_ed25519.pub root@192.168.56.12

k3sup install \
  --ssh-key ~/.ssh/id_ed25519 \
  --ip=192.168.56.11 \
  --user=root \
  --k3s-channel=stable \
  --local-path=k3s-server-1.yaml \
  --context k3s-server-1 \
  --k3s-extra-args="--node-external-ip 192.168.56.11 --node-ip 192.168.56.11"

k3sup join \
  --ssh-key ~/.ssh/id_ed25519 \
  --ip=192.168.56.12 \
  --server-user=root \
  --server-ip=192.168.56.11 \
  --user=root \
  --k3s-channel=stable \
  --k3s-extra-args="--node-external-ip 192.168.56.12 --node-ip 192.168.56.12"


cd /var/tmp

systemctl stop k3s

tar -cvpf k3s-archive-$(date +'%Y-%m-%d-%H-%M').tar \
  -C /var/lib/rancher/k3s server/token server/cred server/tls server/db server/manifests $(ls /var/lib/rancher/k3s/agent/*.{key,crt,kubeconfig} | sed -e 's!/var/lib/rancher/k3s/!!')

systemctl start k3s

k3s-uninstall.sh    # simulate node failure

tar xvpf k3s-archive-2022-02-01-18-12.tar -C /var/lib/rancher/k3s


tar -cvpf k3s-archive-$(date +'%Y-%m-%d-%H-%M').tar \
  -C /var/lib/rancher/k3s server server/cred server/tls server/db server/manifests $(ls /var/lib/rancher/k3s/agent/*.{key,crt,kubeconfig} | sed -e 's!/var/lib/rancher/k3s/!!')
```
