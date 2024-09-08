# Bootstrapping ETCD 

`etcd` is a distributed [key-value store](https://etcd.io) that Kubernetes uses to store all its cluster data -- configurations, state, and secrets. It's the source of truth for the cluster state.

The commands here will be executed on the raspberry pi hosting the control plane, so 
```
export INSTANCE=rp-0

ssh ${INSTANCE}@${INSTANCE}.local
```

*NOTE* In this tutorial we will run `etcd` as a systemd service.

## 1. Download the `etcd` binaries

```
export ETCD_VERSION=${ETCD_VERSION:-v3.5.12}

ARCH=$(uname -m)
if [ "$ARCH" == "x86_64" ]; then
    ARCH="amd64"
elif [ "$ARCH" == "aarch64" ]; then
    ARCH="arm64"
else
    echo "Unsupported architecture: $ARCH"
    exit 1
fi

wget -q --show-progress --https-only --timestamping "https://github.com/etcd-io/etcd/releases/download/${ETCD_VERSION}/etcd-${ETCD_VERSION}-linux-${ARCH}.tar.gz"

tar -xvf "etcd-${ETCD_VERSION}-linux-${ARCH}.tar.gz"

sudo mv "etcd-${ETCD_VERSION}-linux-${ARCH}/etcd"* /usr/local/bin/
```

## 2. Configure `etcd`

```
sudo mkdir -p /etc/etcd /var/lib/etcd
sudo cp ~/ca.pem ~/kubernetes-key.pem ~/kubernetes.pem /etc/etcd/

export INTERNAL_IP=$(ip addr show wlan0 | grep -Po 'inet \K[\d.]+')
export ETCD_NAME=$(hostname -s)
```

## 3. Create the `systemd` service file 

```
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos
[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --data-dir=/var/lib/etcd \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --initial-cluster ${ETCD_NAME}=https://${INTERNAL_IP}:2380 \\
  --initial-cluster-state new \\
  --initial-cluster-token etcd-cluster-0 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --client-cert-auth \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-client-cert-auth \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF
```

## 4. Start it up 

```
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```