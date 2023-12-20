# week 2

## replacing stock network component

- helmchart-controller

## Service LB aka Klipper LB

-> intercepts on port level

- "toy loadbalancer" / "fake LB"
- runs as proxy,
- binds to host port -> if port is used by other LB it stays pending !

- MetalLB
- kube-vip
- OpenELB (prev. Porter) [github.com/openelb/openelb](https://github.com/openelb/openelb)


## Traefik Ingress Controller

-> intercepts on Layer 7

## use different cni

- flannel by default (easy, lightweigt, defaults to VXLAN un-encrypted)
- any othe rcni can be deployed at isntall time (calico, cilium)


cilium -> replaces kube-proxy -> not overlayed -> replaces components !


## definitions

in-tree -> within core k8s codebase

out-of-tree -> part of core k8s repo

=> all drivers are now out-fo-tree, but bundled with upstream k8s



## workload deployment magic

anything dropped into `/var/lib/ranger/k3s/server/manifest`
-> if parsible yaml
-> k3s deployes
-> registers as addon

`kubectl get addon -A`

## helmchart manifests

helmchart CRD


## challenge 4: replace serviceLB

```sh
k3sup install \
  --ssh-key .vagrant/machines/k3s-server-1/virtualbox/private_key \
  --ip=192.168.56.11 \
  --user=root \
  --k3s-channel=stable \
  --local-path=k3s-server-1.yaml \
  --context k3s-node1 \
  --cluster \
  --k3s-extra-args='--disable servicelb --node-external-ip 192.168.56.11 --node-ip 192.168.56.11 --flannel-iface=eth1'

export KUBECONFIG=$(pwd)/k3s-server-1.yaml
```


## challenge 5: change ingress controller

```sh
k3sup install \
  --ssh-key .vagrant/machines/k3s-server-1/virtualbox/private_key \
  --ip=192.168.56.11 \
  --user=root \
  --k3s-channel=stable \
  --local-path=k3s-server-1.yaml \
  --context k3s-server-1 \
  --cluster \
  --k3s-extra-args='--disable traefik --node-external-ip 192.168.56.11 --node-ip 192.168.56.11 --flannel-iface=eth1'

export KUBECONFIG=$(pwd)/k3s-server-1.yaml

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

kubectl create ns ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx


kubectl get pods -n ingress-nginx   # verify installation

curl -I 192.168.56.11 # should return 404 from default backend
```


## challenge 6: change container network interface

```sh
ssh-keygen -R 192.168.56.11 \
&& ssh-keygen -R 192.168.56.12 \
&& ssh-keygen -R 192.168.56.13

ssh-copy-id -i ~/.ssh/id_ed25519.pub root@192.168.56.11 \
  && ssh-copy-id -i ~/.ssh/id_ed25519.pub root@192.168.56.12 \
  && ssh-copy-id -i ~/.ssh/id_ed25519.pub root@192.168.56.13

k3sup install \
  --ssh-key ~/.ssh/id_ed25519 \
  --ip=192.168.56.11 \
  --user=root \
  --k3s-channel=stable \
  --local-path=k3s-server-1.yaml \
  --context k3s-server-1 \
  --k3s-extra-args="--flannel-backend=none --disable-network-policy --node-external-ip 192.168.56.11 --node-ip 192.168.56.11"

k3sup join \
  --ssh-key ~/.ssh/id_ed25519 \
  --ip=192.168.56.12 \
  --server-user=root \
  --server-ip=192.168.56.11 \
  --user=root \
  --k3s-channel=stable \
  --k3s-extra-args="--node-external-ip 192.168.56.12 --node-ip 192.168.56.12"

k3sup join \
  --ssh-key ~/.ssh/id_ed25519 \
  --ip=192.168.56.13 \
  --server-user=root \
  --server-ip=192.168.56.11 \
  --user=root \
  --k3s-channel=stable \
  --k3s-extra-args="--node-external-ip 192.168.56.13 --node-ip 192.168.56.13"


sudo mount bpffs -t bpf /sys/fs/bpf


curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz{,.sha256sum}
```


## challenge 7: deploy apps via helmchart crd

> extension to challenge 5

```sh
ssh-keygen -R 192.168.56.11 \
&& ssh-copy-id -i ~/.ssh/id_ed25519.pub root@192.168.56.11


k3sup install \
  --ssh-key ~/.ssh/id_ed25519 \
  --ip=192.168.56.11 \
  --user=root \
  --k3s-channel=stable \
  --local-path=k3s-server-1.yaml \
  --context k3s-server-1 \
  --cluster \
  --k3s-extra-args='--disable traefik --flannel-iface=eth1 --node-external-ip 192.168.56.11 --node-ip 192.168.56.11'

k3sup install \
  --ip=192.168.34.10 \
  --user=root \
  --k3s-channel=stable \
  --local-path=config.k3s-node1.yaml \
  --context k3s-node1 \
  --cluster \
  --k3s-extra-args='--disable traefik'

# no --flannel-iface=eth1 ??

kubectl apply -f helmchart.yaml

kubectl get helmchart -n kube-system
kubectl get job -n kube-system
kubectl get pod -n kube-system

helm ls -n ingress-nginx
```
