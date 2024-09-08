# Adding another node to your cluster 

To add a new node in your cluster you need to register it to the cluster (and first configure & distribute the necessary credentials) and run the necessary services. It mimics a subset (the non-control plane) part of the work we did for the first raspberry pi.

## 1. Generate worker certficates

```
export INTERNAL_IP=$(ip addr show wlan0  | grep -Po 'inet \K[\d.]+')
export INSTANCE=rp-1

cat > pki/clients/${INSTANCE}-csr.json <<EOF
{
  "CN": "system:node:${INSTANCE}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "${TLS_C}",
      "L": "${TLS_L}",
      "O": "system:nodes",
      "OU": "${TLS_OU}",
      "ST": "${TLS_ST}"
    }
  ]
}
EOF
INTERNAL_IP=$(ip addr show ens3 | grep -Po 'inet \K[\d.]+')
cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -hostname=${INSTANCE},${INTERNAL_IP} \
  -profile=kubernetes \
  pki/clients/${INSTANCE}-csr.json | cfssljson -bare pki/clients/${INSTANCE}
```

## 2. Generate worker kubecofnigs

```
export INSTANCE=rp-1
export KUBERNETES_PUBLIC_ADDRESS=192.168.0.19
export KUBERNETES_CLUSTER_NAME=KallyHomeLab

kubectl config set-cluster ${KUBERNETES_CLUSTER_NAME} \
    --certificate-authority=pki/ca/ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=configs/clients/${INSTANCE}.kubeconfig

kubectl config set-credentials system:node:${INSTANCE} \
    --client-certificate=pki/clients/${INSTANCE}.pem \
    --client-key=pki/clients/${INSTANCE}-key.pem \
    --embed-certs=true \
    --kubeconfig=configs/clients/${INSTANCE}.kubeconfig

kubectl config set-context default \
    --cluster=${KUBERNETES_CLUSTER_NAME} \
    --user=system:node:${INSTANCE} \
    --kubeconfig=configs/clients/${INSTANCE}.kubeconfig

kubectl config use-context default --kubeconfig=configs/clients/${INSTANCE}.kubeconfig
```


## 3. Move them 

```
scp pki/ca/ca.pem \
    pki/clients/${INSTANCE}.pem \
    pki/clients/${INSTANCE}-key.pem \
    pki/proxy/kube-proxy.pem \
    pki/proxy/kube-proxy-key.pem ${INSTANCE}@${INSTANCE}.local:~
```

## 4. Download binaries

```
ssh ${INSTANCE}@${INSTANCE}.local

export K8S_VERSION=1.29.3
export CNI_VERSION=1.4.1
export RUNC_VERSION=1.1.12
export CONTAINERD_VERSION=1.7.14
export CRI_VERSION=1.29.0

ARCH=$(uname -m)
if [ "$ARCH" == "x86_64" ]; then
    ARCH="amd64"
elif [ "$ARCH" == "aarch64" ]; then
    ARCH="arm64"
else
    echo "Unsupported architecture: $ARCH"
    exit 1
fi

wget -q --show-progress --https-only --timestamping \
https://github.com/kubernetes-sigs/cri-tools/releases/download/v${CRI_VERSION}/crictl-v${CRI_CERSION}-linux-${ARCH}.tar.gz \
https://github.com/opencontainers/runc/releases/download/v${RUNC_VERSION}/runc.${ARCH} \
https://github.com/containernetworking/plugins/releases/download/v${CNI_VERSION}/cni-plugins-linux-${ARCH}-v${CNI_VERSION}.tgz \
https://github.com/containerd/containerd/releases/download/v${CONTAINERD_VERSION}/containerd-v${CONTAINERD_VERSION}-linux-${ARCH}.tar.gz \
https://dl.k8s.io/v${K8S_VERSION}/bin/linux/arm64/kube-proxy \
https://dl.k8s.io/v${K8S_VERSION}/bin/linux/arm64/kubelet
```

```
sudo mkdir -p \
 /etc/cni/net.d \
 /opt/cni/bin \
 /var/lib/kubelet \
 /var/lib/kube-proxy \
 /var/lib/kubernetes \
 /var/run/kubernetes
```

```
mkdir containerd
tar -xvf crictl-v${CRI_VERSION}-linux-${ARCH}.tar.gz 
tar -xvf containerd-${CONTAINERD_VERSION}-linux-${ARCH}.tar.gz -C containerd
sudo tar -xvf cni-plugins-linux-${ARCH}-v${CNI_VERSION}.tgz -C /opt/cni/bin/
sudo mv runc.${ARCH} runc
chmod +x crictl kube-proxy kubelet runc 
sudo mv crictl kube-proxy kubelet runc /usr/local/bin/
sudo mv containerd/bin/* /bin/
```

### Pod networking

```
export POD_CIDR=10.200.1.0/24
```

*NOTE:* You need to pick this to be disjoint from your previously picked CIDR blocks.

```
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "1.0.0",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```

```
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{

    "cniVersion": "1.0.0",
    "name": "lo",
    "type": "loopback"
}
EOF
```

## 5. Configure `systemd` services

```
sudo mkdir -p /etc/containerd/
cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    snapshotter = "overlayfs"
    [plugins."io.containerd.grpc.v1.cri".containerd]
      [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
        runtime_type = "io.containerd.runc.v2"
EOF
```


```
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target
[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity
[Install]
WantedBy=multi-user.target
EOF
```

```
sudo cp ~/${HOSTNAME}-key.pem ~/${HOSTNAME}.pem /var/lib/kubelet/
sudo cp ~/${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
sudo cp ~/ca.pem /var/lib/kubernetes/
```

```
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"
EOF
```

```
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service
[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF
```

```
sudo cp kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```

```
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes
[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF
```

## 6. Run

```
sudo systemctl daemon-reload
sudo systemctl enable containerd kubelet kube-proxy
sudo systemctl start containerd kubelet kube-proxy
```

## 7. Check 

```
kubectl get nodes
```
