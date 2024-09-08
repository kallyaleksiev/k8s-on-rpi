# Kubernetes the hard way on a (single) raspberry pi

This tutorial is an updated modification of the legendary [kubernetes-the-hard-way](https://github.com/kelseyhightower/kubernetes-the-hard-way). 

Many people (including myself) have learned a great deal about kubernetes and the more lower-level workings of it via Kelsey's tutorial.

The tutorial here takes more or less the same approach and walks you through setting up your cluster on a single raspberry pi (or any linux box that you have at your disposal for that matter). It contains explanations along the way that might be useful to people just starting out with Kubernetes and it contains (optional) instructions on how to register another node in your cluster.

Note that this is intended as a tutorial to familiarise yourself with certain workings of kubernetes and is not intended for an illustration on how to run kubernetes in production.

To learn more about running kubernetes in production you should consider separating (and separtely scaling the control plane) in a [high availability fashion](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/).

Services like [EKS](https://aws.amazon.com/eks/) and [AKS](https://azure.microsoft.com/en-gb/products/kubernetes-service) abstract the control plane management aspect for you so you can focus on workload management.

If you need to setup and manage the end to end workflows yourself, tools like [`kubeadm`](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/) can be used to automate that including the work in this tutorial.