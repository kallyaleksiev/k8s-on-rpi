# Configuring pod networking

Kubernetes has a [network model](https://kubernetes.io/docs/concepts/services-networking/#the-kubernetes-network-model) and uses it to impose requirements on the exact pods networking implementation. 

The actual logic is delegated to a [CNI-compatible](https://github.com/containernetworking/cni) networking plugin. There are many options to choose from but here we choose to use [calico](https://docs.tigera.io/calico/latest/about/).


## 1. Get calico

```
curl -L https://docs.projectcalico.org/manifests/calico-etcd.yaml -o calico.yaml
```

## 2. Make some changes

*NOTE:* The following command might require modification of the `sed` syntax depending on your operating system.

```
sed -i '' "s/etcd_endpoints: \"http:\/\/<ETCD_IP>:<ETCD_PORT>\"/etcd_endpoints: \"https:\/\/${ETCD_IP}:2379\"/g" calico.yaml
sed -i '' "s/# etcd-cert: null/etcd-cert: $(cat pki\/api\/kubernetes.pem | base64)/g" calico.yaml
sed -i '' "s/# etcd-key: null/etcd-key: $(cat pki\/api\/kubernetes-key.pem | base64)/g" calico.yaml
sed -i '' "s/# etcd-ca: null/etcd-ca: $(cat pki\/ca\/ca.pem | base64)/g" calico.yaml

sed -i '' "s/etcd_ca: \"\"/etcd_ca: \"\/calico-secrets\/etcd-ca\"/g" calico.yaml
sed -i '' "s/etcd_cert: \"\"/etcd_cert: \"\/calico-secrets\/etcd-cert\"/g" calico.yaml
sed -i '' "s/etcd_key: \"\"/etcd_key: \"\/calico-secrets\/etcd-key\"/g" calico.yaml

sed -i '' 's/# - name: CALICO_IPV4POOL_CIDR/- name: CALICO_IPV4POOL_CIDR/g' calico.yaml
sed -i '' 's/#   value: "192.168.0.0\/16"/  value: "10.200.0.0\/16"/g' calico.yaml
```

## 3. Deploy it in your cluster

```
kubectl apply -f calico.yaml
```