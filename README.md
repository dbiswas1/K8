- [Setup Kubernetes Service Mesh Ingress to host microservices using Istio - PART 3](#setup-kubernetes-service-mesh-ingress-to-host-microservices-using-istio---part-3)
  * [Who should read this Blog:](#who-should-read-this-blog-)
  * [Short introduction](#short-introduction)
    + [EKS](#eks)
    + [EKSCTL](#eksctl)
    + [HELM](#helm)
    + [TILLER](#tiller)
    + [ISTIO](#istio)
  * [Problem we are trying to solve](#problem-we-are-trying-to-solve)
  * [Stack used](#stack-used)
  * [Actual implementation](#actual-implementation)
    + [Setup EKSCTL in MAC](#setup-eksctl-in-mac)
    + [Launch EKS](#launch-eks)
    + [Verify EKS Install](#verify-eks-install)
    + [Setup Kubernetes Dashboard](#setup-kubernetes-dashboard)
    + [Setup ISTIO](#setup-istio)
    + [Install Istio with Helm:](#install-istio-with-helm-)
    + [Verify Istio](#verify-istio)
    + [Deploy the Application](#deploy-the-application)
    + [Verify Deployment and Services](#verify-deployment-and-services)

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
6) [TILLER](https://helm.sh/docs/glossary/#release)
7) **ami-0923e4b35a30a5f53** [AmazonLinux2/1.12]


## Actual implementation
Make sure Kubectl is installed. Please refer previous blogs for installing kubectl. also 

### Setup EKSCTL in MAC

1) Make sure aws sdk is configured in mac os
2) Install IAM Authenticator (brew install aws-iam-authenticator)

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
[✔]  saved kubeconfig as "/Users/***********/.kube/config"
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

### Verify EKS Install
```
 master ●  kubectl get nodes
NAME                                           STATUS   ROLES    AGE   VERSION
ip-192-168-42-118.us-west-2.compute.internal   Ready    <none>   32m   v1.12.7
ip-192-168-6-130.us-west-2.compute.internal    Ready    <none>   32m   v1.12.7
ip-192-168-65-11.us-west-2.compute.internal    Ready    <none>   32m   v1.12.7

```
**CLuster Overview**
![Screencap-1](https://github.com/dbiswas1/K8/raw/master/images/EKS-1.png)

**EKS Details**
![Screencap-2](https://github.com/dbiswas1/K8/raw/master/images/EKS-2.png)

**Nodegroups created**
![Screencap-3](https://github.com/dbiswas1/K8/raw/master/images/EKS-3.png)


### Setup Kubernetes Dashboard

```
 master ●  kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
secret/kubernetes-dashboard-certs created
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created

####### Verify Install ########

 master ●  kubectl get pods --namespace kube-system | grep kubernetes-dashboard
kubernetes-dashboard-65c76f6c97-clnkv   1/1     Running   0          5m25s

```

The above dashboard is installed in Private network to access from browser we will use kube proxy(tunneling). This is 
for security reason. However there are options to make it public (not in scope of the blog)
```
kubectl proxy --port=8080 --address='0.0.0.0' --disable-filter=true &

W0616 13:19:43.747800   77245 proxy.go:140] Request filter disabled, your proxy is vulnerable to XSRF attacks, please be cautious


# Then use the following to access the Dashboard
http://localhost:8080/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/overview?namespace=kube-system

# Use toekn to login. to get login use following command

aws-iam-authenticator token -i avm-blog-3 --token-only

k8s-aws-v1.aHR0cHM6Ly9zdHMuYW1hem9uYXdzLmNvbS8_QWN0aW9uPUdldENhbGxlcklkZW50aXR5JlZlcnNpb249MjAxMS0wNi0xNSZYLUFtei1BbG
dvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFVTlFHTktPWTcyN0pMS00yJTJGMjAxOTA2MTYlMkZ1cy1lYXN0LTElMkZzd
HMlMkZhd3M0X3JlcXVlc3QmWC1BbXotRGF0ZT0yMDE5MDYxNlQ
```
**Dashboard**
![]((https://github.com/dbiswas1/K8/raw/master/images/EKS-4.png))

### Setup ISTIO
1) Download Istio and change directory
```
 master ●  curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.1.8 sh -

Downloading istio-1.1.8 from https://github.com/istio/istio/releases/download/1.1.8/istio-1.1.8-osx.tar.gz ...  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   612    0   612    0     0   1215      0 --:--:-- --:--:-- --:--:--  1214
100 14.1M  100 14.1M    0     0   613k      0  0:00:23  0:00:23 --:--:-- 1872k
Istio 1.0.8 Download Complete!

Istio has been successfully downloaded into the istio-1.1.8 folder on your system.


# Setup the istoctl path and verify

 master ●  cd istio-1.1.8

 master ●✚  export PATH=$PATH:/Users/***********/******/code/K8/istio-1.1.8/bin
 
 master ●✚  istioctl version
Version: 1.0.8
GitRevision: 11b640bb11593138a904f81e572d40e5e70b089b
User: root@d76cad0a-8935-11e9-887b-0a580a2c0403
Hub: docker.io/istio
GolangVersion: go1.10.4
BuildStatus: Clean

# Add Helm Repo Chart
 master ●✚  helm repo add istio.io https://storage.googleapis.com/istio-release/releases/1.1.8/charts/

"istio.io" has been added to your repositories
```

### Install Istio with Helm:
Make sure you are in istio directory as most of the sample/k8 objects are in the install folder and have helm installed

Refer: https://helm.sh/docs/using_helm/#installing-helm for installing helm

```
# Create Istio Namespace.
 master ●✚  kubectl create namespace istio-system

#Install all the Istio Custom Resource Definitions (CRDs) using kubectl apply, and wait a few seconds for the CRDs 
to be committed in the Kubernetes API-server:

 master ●✚   master ●✚  helm template install/kubernetes/helm/istio-init --name istio-init --namespace istio-system | kubectl apply -f -
              
              configmap/istio-crd-10 created
              configmap/istio-crd-11 created
              serviceaccount/istio-init-service-account created
              clusterrole.rbac.authorization.k8s.io/istio-init-istio-system unchanged
              clusterrolebinding.rbac.authorization.k8s.io/istio-init-admin-role-binding-istio-system unchanged
              job.batch/istio-init-crd-10 created
              job.batch/istio-init-crd-11 created
              
# Verify CRD's it should be 53

 master ●✚  kubectl get crds | grep 'istio.io\|certmanager.k8s.io' | wc -l
      53

#Install Default Profile for Prod deployments

  master ●✚  helm template install/kubernetes/helm/istio --name istio --namespace istio-system | kubectl apply -f -
 
 ############### Curtailed Output ###############
 
poddisruptionbudget.policy/istio-galley created
poddisruptionbudget.policy/istio-ingressgateway created
poddisruptionbudget.policy/istio-policy created
poddisruptionbudget.policy/istio-telemetry created
poddisruptionbudget.policy/istio-pilot created
configmap/istio-galley-configuration created
----
----
-----
rule.config.istio.io/promtcpconnectionopen created
rule.config.istio.io/promtcpconnectionclosed created
handler.config.istio.io/kubernetesenv created
rule.config.istio.io/kubeattrgenrulerule created
rule.config.istio.io/tcpkubeattrgenrulerule created
kubernetes.config.istio.io/attributes created
destinationrule.networking.istio.io/istio-policy created
destinationrule.networking.istio.io/istio-telemetry created
############### Curtailed Output ###############

```

### Verify Istio

Verify the Configuration you elected all exist and pods in istio-system namespace

```
 master ●✚  kubectl get svc -n istio-system
NAME                     TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)                                                                                                                                      AGE
istio-citadel            ClusterIP      10.100.79.162    <none>                                                                   8060/TCP,15014/TCP                                                                                                                           2m20s
istio-galley             ClusterIP      10.100.111.160   <none>                                                                   443/TCP,15014/TCP,9901/TCP                                                                                                                   2m26s
istio-ingressgateway     LoadBalancer   10.100.117.186   ab54d0978901411e99bd2023cce26310-454040857.us-west-2.elb.amazonaws.com   15020:32150/TCP,80:31380/TCP,443:31390/TCP,31400:31400/TCP,15029:32251/TCP,15030:30236/TCP,15031:30552/TCP,15032:31444/TCP,15443:31327/TCP   2m25s
istio-pilot              ClusterIP      10.100.217.101   <none>                                                                   15010/TCP,15011/TCP,8080/TCP,15014/TCP                                                                                                       2m22s
istio-policy             ClusterIP      10.100.28.69     <none>                                                                   9091/TCP,15004/TCP,15014/TCP                                                                                                                 2m24s
istio-sidecar-injector   ClusterIP      10.100.126.132   <none>                                                                   443/TCP                                                                                                                                      2m20s
istio-telemetry          ClusterIP      10.100.47.126    <none>                                                                   9091/TCP,15004/TCP,15014/TCP,42422/TCP                                                                                                       2m23s
prometheus               ClusterIP      10.100.228.73    <none>                                                                   9090/TCP                                                                                                                                     2m21s


 master ●✚  kubectl get pods -n istio-system
NAME                                      READY   STATUS      RESTARTS   AGE
istio-citadel-7dbf78bf8f-f5vlt            1/1     Running     0          2m28s
istio-cleanup-secrets-1.1.8-w2872         0/1     Completed   0          2m58s
istio-galley-7f874545bd-blcjc             1/1     Running     0          2m34s
istio-ingressgateway-75479dbb99-vqz6n     1/1     Running     0          2m33s
istio-init-crd-10-js7zg                   0/1     Completed   0          5m56s
istio-init-crd-11-4b4ch                   0/1     Completed   0          5m56s
istio-pilot-6b488d9f66-fmc8f              2/2     Running     0          2m30s
istio-policy-6b5fbbb7bf-fvwnj             2/2     Running     1          2m32s
istio-security-post-install-1.1.8-8xm8f   0/1     Completed   0          2m55s
istio-sidecar-injector-5c77d99d8-7jgl4    1/1     Running     0          2m28s
istio-telemetry-5c9b4d6b95-tn7t8          2/2     Running     1          2m31s
prometheus-5977597c75-prl8l               1/1     Running     0          2m29s
```

### Deploy the Application
After Istio is setup we will install one of the samples which is part of ISTIO download

```
 master ●✚  kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml)
service/details created
deployment.extensions/details-v1 created
service/ratings created
deployment.extensions/ratings-v1 created
service/reviews created
deployment.extensions/reviews-v1 created
deployment.extensions/reviews-v2 created
deployment.extensions/reviews-v3 created
service/productpage created
deployment.extensions/productpage-v1 created


# Define the ingress gateway

 master ●✚   kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
gateway.networking.istio.io/bookinfo-gateway created
virtualservice.networking.istio.io/bookinfo created

# Verify the Gateway

 master ●✚  kubectl get gateway
NAME               AGE
bookinfo-gateway   26s


```

### Verify Deployment and Services

The application will be in default gateway

```
 master ●✚  kubectl get services
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.100.149.13    <none>        9080/TCP   4m36s
kubernetes    ClusterIP   10.100.0.1       <none>        443/TCP    135m
productpage   ClusterIP   10.100.51.72     <none>        9080/TCP   4m31s
ratings       ClusterIP   10.100.64.106    <none>        9080/TCP   4m34s
reviews       ClusterIP   10.100.194.249   <none>        9080/TCP   4m33s
```

Now to view the application, get the loadbalancer address( external ip) of istio-ingressgateway
and paste http://<external-ip>/productpage see below
```
 master ●✚  kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)                                                                                                                                      AGE
istio-ingressgateway   LoadBalancer   10.100.117.186   ab54d0978901411e99bd2023cce26310-454040857.us-west-2.elb.amazonaws.com   15020:32150/TCP,80:31380/TCP,443:31390/TCP,31400:31400/TCP,15029:32251/TCP,15030:30236/TCP,15031:30552/TCP,15032:31444/TCP,15443:31327/TCP   11m
```

![](https://github.com/dbiswas1/K8/raw/master/images/EKS-5.png)

