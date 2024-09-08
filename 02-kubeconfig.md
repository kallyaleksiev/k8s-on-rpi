# Generating `kubeconfig` files 

A [kubeconfig file](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) is a configuration file used by Kubernetes to manage access to different clusters. It contains information such as the cluster's API server address and user credentials. It allows various clients to interact with the kube apiserver.

In this section we generate the various kubeconfig files that will be needed to interact with the cluster.

## 1. Generate worker kubeconfigs

Each worker node in the Kubernetes cluster needs a kubeconfig file to authenticate with the API server. This file will be used by the node's `kubelet`.

```
export INSTANCE=rp-0
export KUBERNETES_PUBLIC_ADDRESS=192.168.0.20
# Find it for example using the technique from the 01-pki.md
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

## 2. Generate kube-proxy kubeconfig

```
kubectl config set-cluster ${KUBERNETES_CLUSTER_NAME} \
  --certificate-authority=pki/ca/ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=configs/proxy/kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=pki/proxy/kube-proxy.pem \
  --client-key=pki/proxy/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=configs/proxy/kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=${KUBERNETES_CLUSTER_NAME} \
  --user=system:kube-proxy \
  --kubeconfig=configs/proxy/kube-proxy.kubeconfig
kubectl config use-context default --kubeconfig=configs/proxy/kube-proxy.kubeconfig
```

## 3. Generate `kube-controller-manager` kubeconfig

```
kubectl config set-cluster ${KUBERNETES_CLUSTER_NAME} \
  --certificate-authority=pki/ca/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=configs/controller/kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=pki/controller/kube-controller-manager.pem \
  --client-key=pki/controller/kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=configs/controller/kube-controller-manager.kubeconfig

kubectl config set-context default \
  --cluster=KallyHomeLab \
  --user=system:kube-controller-manager \
  --kubeconfig=configs/controller/kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=configs/controller/kube-controller-manager.kubeconfig
```

## 4. Generate `kube-scheduler` kubeconfig

```
kubectl config set-cluster KallyHomeLab \
  --certificate-authority=pki/ca/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=configs/scheduler/kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=pki/scheduler/kube-scheduler.pem \
  --client-key=pki/scheduler/kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=configs/scheduler/kube-scheduler.kubeconfig

kubectl config set-context default \
  --cluster=KallyHomeLab \
  --user=system:kube-scheduler \
  --kubeconfig=configs/scheduler/kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=configs/scheduler/kube-scheduler.kubeconfig
```

## 5. Generate admin user kubeconfig

This will be the actual user client admin (i.e. persona writing `kubectl` commands) and the components that persona shares this with (e.g. scripts etc).

```
kubectl config set-cluster ${KUBERNETES_CLUSTER_NAME} \
  --certificate-authority=pki/ca/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=configs/admin/admin.kubeconfig

kubectl config set-credentials admin \
  --client-certificate=pki/admin/admin.pem \
  --client-key=pki/admin/admin-key.pem \
  --embed-certs=true \
  --kubeconfig=configs/admin/admin.kubeconfig

kubectl config set-context default \
  --cluster=${KUBERNETES_CLUSTER_NAME} \
  --user=admin \
  --kubeconfig=configs/admin/admin.kubeconfig

kubectl config use-context default --kubeconfig=configs/admin/admin.kubeconfig
```

## 6. Move the kubeconfigs

```
scp configs/clients/${INSTANCE}.kubeconfig \
    configs/proxy/kube-proxy.kubeconfig \
    ${INSTANCE}@${INSTANCE}.local:~/

scp configs/admin/admin.kubeconfig \
    configs/controller/kube-controller-manager.kubeconfig \
    configs/scheduler/kube-scheduler.kubeconfig \
    ${INSTANCE}@${INSTANCE}.local:~/
```


