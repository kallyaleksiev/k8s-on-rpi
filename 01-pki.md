# PKI

The first thing that is needed is to bootstrap your PKI infra.

Public Key Infrastructure (PKI) is a collection of encryption keys, certificates, and tools, essential for securing communications within your Kubernetes cluster. The idea is to allow for secure communication between the various components of the Kubernetes system (e.g. kubelet <> apiserver, client <> apiserver)

In this step, we create a bunch of certificates and keys and put them in the right directories.

```
mkdir -p pki/{admin,api,ca,clients,controller,front-proxy,proxy,scheduler,service-account,users}

export TLS_C="GB"
export TLS_L="LONDON"
export TLS_OU="KallyHomeLab"
export TLS_ST="LONDON"
```

## 1. Generate CA cert
The Certificate Authority (CA) is the root of trust in your PKI infrastructure. It is responsible for signing and issuing all other certificates within the cluster.

```
cat > pki/ca/ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
cat > pki/ca/ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "${TLS_C}",
      "L": "${TLS_L}",
      "O": "Kubernetes",
      "OU": "${TLS_OU}",
      "ST": "${TLS_ST}"
    }
  ]
}
EOF
cfssl gencert -initca pki/ca/ca-csr.json | cfssljson -bare pki/ca/ca
```

*NOTE:* For full spec of the json configs for certificates check out the [following code](https://github.com/cloudflare/cfssl/blob/12a0addeae31a6bf4c49afbda9531411a628e0f9/config/config.go#L78).

## 2. Generate admin credentials (to be used with`kubectl`)

The admin certificate and key are required for using `kubectl` to interact with your Kubernetes cluster.

The admin credentials are given the highest level of access within the cluster, usually mapped to the system:masters [group](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).

```
cat > pki/admin/admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "${TLS_C}",
      "L": "${TLS_L}",
      "O": "system:masters",
      "OU": "${TLS_OU}",
      "ST": "${TLS_ST}"
    }
  ]
}
EOF

cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -profile=kubernetes \
  pki/admin/admin-csr.json | cfssljson -bare pki/admin/admin
```

## 3. Generate worker(s) credentials

Each Kubernetes worker node might make requests to the apiserver via its kubelet. This requires a certificate to authenticate with the cluster. The Node Authorizer component in Kubernetes checks the credentials of each node to ensure they are legitimate and allowed to perform their role within the cluster.

The `kubelet` is the service that runs on each worker node and is responsible for registering that node with the Kubernetes apiserver and for managing the state of all pods on that node. Read more about the kubelet [here](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/).

On the raspberry pi find out
```
ip addr show wlan0  | grep -Po 'inet \K[\d.]+'
```

and set it as `INTERNAL_IP` on the host alongside:

```
export INTERNAL_IP=...
export INSTANCE=rp-0
```

```
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

cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -hostname=${INSTANCE},${INTERNAL_IP} \
  -profile=kubernetes \
  pki/clients/${INSTANCE}-csr.json | cfssljson -bare pki/clients/${INSTANCE}
```

## 4. Generate the `kube-controller-manager` credentials
The `kube-controller-manager` is a core component of Kubernetes that runs the basic packaged kubernetes controllers (control loops). Read more about its worksing [here](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/).

```
cat > pki/controller/kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "${TLS_C}",
      "L": "${TLS_L}",
      "O": "system:kube-controller-manager",
      "OU": "${TLS_OU}",
      "ST": "${TLS_ST}"
    }
  ]
}
EOF
cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -profile=kubernetes \
  pki/controller/kube-controller-manager-csr.json | cfssljson -bare pki/controller/kube-controller-manager
```

## 5. Generate the `kube-proxy` credentials

The `kube-proxy` maintains network rules on each node, allowing network communication to your Pods. You can read more about it [here](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/).

```
cat > pki/proxy/kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "${TLS_C}",
      "L": "${TLS_L}",
      "O": "system:node-proxier",
      "OU": "${TLS_OU}",
      "ST": "${TLS_ST}"
    }
  ]
}
EOF
cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -profile=kubernetes \
  pki/proxy/kube-proxy-csr.json | cfssljson -bare pki/proxy/kube-proxy
```

## 6. Generate the `kube-scheduler` credentials

The `kube-scheduler` is responsible for scheduling pods on available nodes. You can read more about it [here](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/).

```
cat > pki/scheduler/kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "${TLS_C}",
      "L": "${TLS_L}",
      "O": "system:kube-scheduler",
      "OU": "${TLS_OU}",
      "ST": "${TLS_ST}"
    }
  ]
}
EOF
cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -profile=kubernetes \
  pki/scheduler/kube-scheduler-csr.json | cfssljson -bare pki/scheduler/kube-scheduler
```

## 7. Generate `kube-apiserver` credentials

The `kube-apiserver` is the interface for the Kubernetes control plane. It exposes the Kubernetes API to clients and communicates the cluster's state to them. You can read more about it [here](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/).

```
cat > pki/api/kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "${TLS_C}",
      "L": "${TLS_L}",
      "O": "Kubernetes",
      "OU": "${TLS_OU}",
      "ST": "${TLS_ST}"
    }
  ]
}
EOF
export KUBERNETES_PUBLIC_ADDRESS=$INTERNAL_IP
export KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local
cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -hostname=10.32.0.1,127.0.0.1,kubernetes.default,${KUBERNETES_PUBLIC_ADDRESS},${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes pki/api/kubernetes-csr.json | cfssljson -bare pki/api/kubernetes
```

*NOTE:* The -hostname parameter includes several IP addresses and DNS names that the API server should be able to respond to. The IP 10.32.0.1 is the default service IP for the Kubernetes API and it is required to ensure the API server can be accessed correctly within the cluster. The `zINTERNAL_IP` is the internal network IP for the raspberry pi where the control plane will be situated, e.g. in my case `192.168.0.20`. 

## 8. Generate the service accounts credentials

[Service accounts](https://kubernetes.io/docs/concepts/security/service-accounts/) are used by Kubernetes Pods to interact with the API server. A service account token is a signed JWT (JSON Web Token) that the API server uses to authenticate requests made by the service accounts.

```
cat > pki/service-account/service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "${TLS_C}",
      "L": "${TLS_L}",
      "O": "Kubernetes",
      "OU": "${TLS_OU}",
      "ST": "${TLS_ST}"
    }
  ]
}
EOF
cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -profile=kubernetes \
  pki/service-account/service-account-csr.json | cfssljson -bare pki/service-account/service-account
```

## 9. Move the certificates 

```
scp pki/ca/ca.pem pki/clients/${INSTANCE}-key.pem \
  pki/clients/${INSTANCE}.pem ${INSTANCE}@${INSTANCE}.local:~/

scp pki/ca/ca.pem \
  pki/ca/ca-key.pem \
  pki/api/kubernetes-key.pem \
  pki/api/kubernetes.pem pki/service-account/service-account-key.pem \
  pki/service-account/service-account.pem pki/front-proxy/front-proxy-key.pem \
  pki/front-proxy/front-proxy.pem ${INSTANCE}@${INSTANCE}.local:~/
```