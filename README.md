# Setup Kubernetes Service Mesh Ingress to host microservices using Istio - PART 3

This is Part 3 of the Blog series we have started ([Part-1](https://blog.avmconsulting.net/posts/2019-04-09-setup-kubernetes-cluster-with-terraform-and-kops-part-1/) and [Part-2](https://blog.avmconsulting.net/posts/2019-04-29-setup-kubernetes-cluster-with-terraform-and-kops-build-enterprise-containers-ready-using-part-2/))
Previous blogs where more about Setting up Cluster and Creating Docker images. we wil see in this Blog how a typical microservice is deployed in 
K8 service mesh using ISTIO.

## Who should read this Blog:
This Blog will be quick start guide to get the basic understanding of service mesh ISTIO. Wil be using 
Book Info application to deploy and check the varipus out of box feature of ISTIO

## Short introduction
This is not an advanced level tutorial. 
1) I am using EKS to to setup the K8 Cluster because :
    1) Creates HA control Plane
    2) IAM integration out of the box
    3) Certificate Management 
    4) setup LB
2) Using **AWS** as cloud provider 
3) Using Helm for deployment

### EKS
Amazon EKS runs the Kubernetes management infrastructure for you across multiple AWS availability zones to eliminate a
single point of failure. Amazon EKS is certified Kubernetes conformant so you can use existing tooling and plugins from
partners and the Kubernetes community. Applications running on any standard Kubernetes environment are fully compatible
and can be easily migrated to Amazon EKS.

REF: https://aws.amazon.com/eks/
 
### EKSCTL 
eksctl is a simple CLI tool for creating clusters on EKS - Amazon's new managed Kubernetes service for EC2. 
It is written in Go, and uses CloudFormation.

REF: https://github.com/weaveworks/eksctl

### HELM
Helm helps you manage Kubernetes applications — Helm Charts help you define, install, and upgrade even the most complex 
Kubernetes application.

REF: https://helm.sh/

### ISTIO
Istio is a platform which helps in service discovery, managing and connecting microservices, Canary and A/B testing. 
The other out of the box features provided by Istio are :
1) [Grafana](https://grafana.com/): Analytics and monitoring of services in the cluster.
2) [Promethues](https://prometheus.io/): Used for collecting the metrics from the cluster at regular interval.

For more detailed explanation : https://istio.io/docs/concepts/what-is-istio/overview/

## Problem we are trying to solve
We are trying to demonstrate deployment of micrioservices in service mesh and understand how microsercvices communicate
with each other using the ingress in K8 Cluster. We will Explore and see what are out of box from ISTIO which can reduce
more OPEX.

## Stack used
1) [Helm](https://helm.sh/)
2) [ISTIO](https://istio.io)
3) [ENVSUBST or GETTEXT](https://github.com/tardate/LittleCodingKata/tree/master/tools/envsubst)
4) [EKS](https://aws.amazon.com/eks/)
5) [EKSCTL](https://github.com/weaveworks/eksctl)
5) **ami-0923e4b35a30a5f53** [AmazonLinux2/1.12]


## Actual implementation
Make sure Kubectl is installed. Please refer previous blogs for installing kubectl. also 

### Setup EKSCTL in MAC

1) Make sure aws sdk is configured in mac os
```
 master ●✚   curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

 master ●✚   mv /tmp/eksctl /usr/local/bin

# Verify Install
  master ●✚  eksctl version
[ℹ]  version.Info{BuiltAt:"", GitCommit:"", GitTag:"0.1.35"}

```

### Launch EKS
This may take some time and will create 
VPC
Subnets
Certifcates
API endpoint and Load balancer

```
  master ●✚  eksctl create cluster \
--name=avm-blog-3 \
--version=1.12 \
--node-type=t3.medium \
--nodes=3 \
--nodes-min=1 \
--nodes-max=4 \
--node-ami=auto

###################### Output ###############################################
[ℹ]  using region us-west-2
[ℹ]  setting availability zones to [us-west-2c us-west-2a us-west-2b]
[ℹ]  subnets for us-west-2c - public:192.168.0.0/19 private:192.168.96.0/19
[ℹ]  subnets for us-west-2a - public:192.168.32.0/19 private:192.168.128.0/19
[ℹ]  subnets for us-west-2b - public:192.168.64.0/19 private:192.168.160.0/19
[ℹ]  nodegroup "ng-a1c7c70d" will use "ami-0923e4b35a30a5f53" [AmazonLinux2/1.12]
[ℹ]  creating EKS cluster "avm-blog-3" in "us-west-2" region
[ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial nodegroup
[ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=us-west-2 --name=avm-blog-3'
[ℹ]  2 sequential tasks: { create cluster control plane "avm-blog-3", create nodegroup "ng-a1c7c70d" }
[ℹ]  building cluster stack "eksctl-avm-blog-3-cluster"
[ℹ]  deploying stack "eksctl-avm-blog-3-cluster"
[ℹ]  building nodegroup stack "eksctl-avm-blog-3-nodegroup-ng-a1c7c70d"
[ℹ]  deploying stack "eksctl-avm-blog-3-nodegroup-ng-a1c7c70d"
[✔]  all EKS cluster resource for "avm-blog-3" had been created
[✔]  saved kubeconfig as "/Users/deepakkumar.biswas/.kube/config"
[ℹ]  adding role "arn:aws:iam::303882392497:role/eksctl-avm-blog-3-nodegroup-ng-a1-NodeInstanceRole-1E0WFDTJ9JHL6" to auth ConfigMap
[ℹ]  nodegroup "ng-a1c7c70d" has 0 node(s)
[ℹ]  waiting for at least 1 node(s) to become ready in "ng-a1c7c70d"
[ℹ]  nodegroup "ng-a1c7c70d" has 3 node(s)
[ℹ]  node "ip-192-168-42-118.us-west-2.compute.internal" is not ready
[ℹ]  node "ip-192-168-6-130.us-west-2.compute.internal" is ready
[ℹ]  node "ip-192-168-65-11.us-west-2.compute.internal" is not ready
[✖]  neither aws-iam-authenticator nor heptio-authenticator-aws are installed
[ℹ]  cluster should be functional despite missing (or misconfigured) client binaries
[✔]  EKS cluster "avm-blog-3" in "us-west-2" region is ready

###################### Output ###############################################

```
! [Screencap-1](https://github.com/dbiswas1/K8/blob/master/images/EKS-1.png)