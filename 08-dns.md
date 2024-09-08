# Set up DNS service discovery

In order to be able to contact services by their (DNS) names you need to have a [DNS solution](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) in your cluster. In this case we will use [Core DNS](https://coredns.io/).

## 1. Download CoreDNS

```
curl https://raw.githubusercontent.com/coredns/deployment/master/kubernetes/coredns.yaml.sed -o coredns.yaml.sed
curl https://raw.githubusercontent.com/coredns/deployment/master/kubernetes/deploy.sh -o deploy.sh
```

## 2. Apply

```
bash ./deploy.sh -s -i 10.32.0.1 > coredns.yaml
kubectl apply -f coredns.yaml
```

## 3. Test DNS

```
kubectl exec -i -t dnsutils -- nslookup kubernetes.default
```