# week 1

## uninstalling k3s

```sh
/usr/local/bin/k3s-uninstall.sh        # from a server node

/usr/local/bin/k3s-agent-uninstall.sh  # from an agent node
```

[rancher.com/docs/k3s/latest/en/installation/uninstall](https://rancher.com/docs/k3s/latest/en/installation/uninstall/)

## challenge 1 - install k3s single server

```sh
IP="$(ip -4 addr show eth1 | grep -oP "(?<=inet ).*(?=/)")"

curl -sfL https://get.k3s.io | K3S_NODE_NAME=$(hostname) INSTALL_K3S_EXEC="--node-external-ip $IP --node-ip $IP" sh -

cat /var/lib/rancher/k3s/server/node-token

TOKEN="K102d30dc6ee99b5381eacfb51e88157544e46f8ad2a0ae06a238fafbd559fb7a2a::server:1a34ac735c8503b1e475223369bdc808"

curl -sfL https://get.k3s.io | K3S_URL="https://192.168.56.11:6443" K3S_TOKEN="$TOKEN" sh -
```


## challenge 2 - install k3s multi server + embedded etcd

```sh
export IP="$(ip -4 addr show eth1 | grep -oP "(?<=inet ).*(?=/)")" && \
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --cluster-init --node-ip $IP --node-external-ip $IP" sh -s - && \
cat /var/lib/rancher/k3s/server/token


export \
  IP="$(ip -4 addr show eth1 | grep -oP "(?<=inet ).*(?=/)")" \
  MASTER_IP="192.168.56.11" \
  TOKEN="K105e57f51f147df610627dad77331913e2b7138a5a9d6580641a54c9eb0345eeaf::server:3ae3e6309ef932200916c4a314ede4a1" && \
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --server https://${MASTER_IP}:6443 --node-ip $IP --node-external-ip $IP" \
  K3S_TOKEN="$TOKEN" \
  K3S_NODE_NAME="$(hostname)" sh -
```


## challenge 3 - install k3s multi server + external datastore

```sh
zypper install docker \
  && systemctl enable docker \
  && systemctl start docker \
  && usermod -G docker -a $USER
  && docker info

docker run -d \
  --name mariadb \
  -e MYSQL_ROOT_PASSWORD="s00per" \
  -e MYSQL_DATABASE="kubernetes" \
  -e MYSQL_USER="k3s" \
  -e MYSQL_PASSWORD="d00per" \
  -p 3306:3306 \
  -v /opt/mariadb:/var/lib/mysql \
  mariadb:10.3


mysql -h 192.168.56.11 -P 3306 --protocol=TCP -u root -p

"mysql://k3s:d00per@tcp(192.168.56.11:3306)/kubernetes"



export IP="$(ip -4 addr show eth1 | grep -oP "(?<=inet ).*(?=/)")" && \
curl -sfL https://get.k3s.io | \
  K3S_DATASTORE_ENDPOINT="mysql://k3s:d00per@tcp($IP:3306)/kubernetes" \
  INSTALL_K3S_EXEC="server --node-ip $IP --node-external-ip $IP" sh -s -


export IP="$(ip -4 addr show eth1 | grep -oP "(?<=inet ).*(?=/)")" \
  DATASTORE_IP="192.168.56.11" \
  TOKEN="K103b56afed61b5e8ae0e6cd125ded8512215c1026fd759e77a90c0aec6f2a0ec9b::server:7c7f85e34b2a1c5abda0fa5a2ee59830" && \
curl -sfL https://get.k3s.io | \
  K3S_DATASTORE_ENDPOINT="mysql://k3s:d00per@tcp($DATASTORE_IP:3306)/kubernetes" \
  K3S_TOKEN="$TOKEN" \
  INSTALL_K3S_EXEC="server --node-ip $IP --node-external-ip $IP" sh -s -
```
