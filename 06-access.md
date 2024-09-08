# Set-up access

## 1. Generate some more kubeconfigs

```
# use the technique in section 01-pki.md to find this ip address 
export KUBERNETES_PUBLIC_ADDRESS=192.168.0.20
export KUBERNETES_CLUSTER_NAME=KallyHomeLab

kubectl config set-cluster ${KUBERNETES_CLUSTER_NAME} \
        --certificate-authority=pki/ca/ca.pem \
        --embed-certs=true \
        --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
        --kubeconfig=configs/admin/admin-remote.kubeconfig

kubectl config set-credentials admin \
        --client-certificate=pki/admin/admin.pem \
        --client-key=pki/admin/admin-key.pem \
        --embed-certs=true \
        --kubeconfig=configs/admin/admin-remote.kubeconfig

kubectl config set-context ${KUBERNETES_CLUSTER_NAME} \
        --cluster=${KUBERNETES_CLUSTER_NAME} \
        --user=admin \
        --kubeconfig=configs/admin/admin-remote.kubeconfig

kubectl config use-context ${KUBERNETES_CLUSTER_NAME} --kubeconfig=configs/admin/admin-remote.kubeconfig

cp configs/admin/admin-remote.kubeconfig ~/.config/${USER}/${KUBERNETES_CLUSTER_NAME}.kubeconfig
```

Now if you run the following command you should see your control plane node:

```
kubectl get nodes
```