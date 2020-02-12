---
author: "Dimitris Traskas"
date: 2019-11-26
linktitle: Kubernetes cluster provisioning with eksctl
menu:
  main:
    parent: 
next: 
prev: 
title: Kubernetes cluster provisioning with eksctl
weight: 10
summary: "Using eksctl to provision a brand new Kubernetes cluster. Start a new cluster from scratch and set up a K8S dashboard, storage classes, Helm and more."
BookToC: true
---

Provisioning and managing production Kubernetes clusters in platforms such as AWS was never without it's problems and compared to Google Cloud there are still a lot of things AWS can improve with. When we first started setting up Kubernetes in production we used [Kops](https://kops.sigs.k8s.io/). Kops is a great CLI tool that allows you to go from zero to hero in a matter of a few minutes. With the advent of AWS managed Kubernetes clusters, we realised that there might be some benefits going down the route of using the bespoke platform tools. 

The [EKS](https://aws.amazon.com/eks/) service for provisioning Kubernetes in AWS is pretty basic in its functionality so we needed to use something closer to Kops. Using [eksctl](https://eksctl.io/), the official AWS CLI tool for Kubernetes provisioning and cluster management was the solution. The `eksctl` tool was originally created by **Weaveworks** but later adopted by AWS. With it you can create your cluster, nodegroups, set up VPC networking, auto scaling groups, manage IAM users and roles, and more. Once you have a cluster up and running you can make a few changes but still not to the level of what Kops gives you. From what we have seen so far `eksctl` is a good tool but there is still a lot of work that needs to be done.

In this short guide we will go through the process of setting up a small cluster of 3 workers that are used as web servers. We will also install the **Helm** deployment tool, Kubernetes dasbhoard, an NGINX Ingress Controller, one storage class that could be used to provision encrypted EBS volumes on AWS and Prometheus/Grafana for monitoring. 


## Create a new Cluster

We will get started with the hello world of Kubernetes clusters, a 3 node cluster called `Alice`. The cluster setup of course requires the use of the `eksctl` tool that you can download from [here](https://eksctl.io/introduction/installation/). Once the tool is downloaded the process requires the deployment of the new cluster using the `create cluster` command as seen below. 

```
eksctl create cluster -f cluster.yaml
```

If you are using an AWS profile then you will need to execute the command above with the profile specified.

```
eksctl create cluster -f cluster.yaml --profile my-profile
```

The `Alice` yaml file essentially defines the Cloudformation stacks that will get deployed once `eksctl` is executed. Below we are setting up the absolute minimum by using the cluster-autoscaler, some basic attached policies and a 3 node nodegroup.

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: alice
  region: eu-west-1

iam:
  serviceRoleARN: "arn:aws:iam::322604721144:role/eksServiceRole"
  withOIDC: true
  serviceAccounts:  
  - metadata:
      name: cluster-autoscaler
      namespace: kube-system
      labels: {aws-usage: "cluster-ops"}
    attachPolicy: # inline policy can be defined along with `attachPolicyARNs`
      Version: "2012-10-17"
      Statement:
      - Effect: Allow
        Action:
        - "autoscaling:DescribeAutoScalingGroups"
        - "autoscaling:DescribeAutoScalingInstances"
        - "autoscaling:DescribeLaunchConfigurations"
        - "autoscaling:DescribeTags"
        - "autoscaling:SetDesiredCapacity"
        - "autoscaling:TerminateInstanceInAutoScalingGroup"
        Resource: '*'

nodeGroups:
  - name: web-servers
    instanceType: m4.large
    minSize: 2
    maxSize: 3
    desiredCapacity: 3
    amiFamily: AmazonLinux2
    ami: auto
    volumeSize: 10
    availabilityZones: ["eu-west-2a", "eu-west-2b", "eu-west-2c"]
    labels:
      nodegroup-type: web-servers
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
      withAddonPolicies:
        autoScaler: true
        imageBuilder: true
    tags:
      # EC2 tags required for cluster-autoscaler auto-discovery
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/img-prod2: "owned"
```


It will take **15-20** minutes to create the new cluster using `eksctl`, which essentially deploys a new VPC and subnets, new nodegroups and service accounts for access to relevant AWS services, and of course the EKS cluster.

Once the cluster is ready and running you will be able to see all the nodes joining and at `<ready>` mode. The `eksctl` tool sets up automatically access to the cluster by modifying the `./kube/config` file so it is not required to do anything further for access.


## Set up Helm for deployments

You will then need to setup Helm for deploying your databases, applications and services, by creating a new service account and applying the following:

```
kubectl create clusterrolebinding add-on-cluster-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:default
```

Once that is done you will to execute the command below to install the Tiller in Kubernetes.

```
helm init 
```
and then this to check things are running:
```
helm ls
```


## Set up a Kubernetes Dashboard for managing your cluster

You will need to follow the instructions [here](https://docs.aws.amazon.com/eks/latest/userguide/dashboard-tutorial.html) to install the metrics server first. Once you deploy the metrics server, you can deploy the Kubernetes dashboard by running:

```
kubectl apply -f kubernetes-dashboard.yaml
```

Then create a new service account by executing this:

```
kubectl apply -f eks-admin-service-account.yaml
```

## Set up an AWS Storage class

The command below can be used to add a new Storage class. In this example we are adding a gp2-encrypted storage class which creates the specific type of EBS volumes.

```
kubectl apply -f gp2-encrypted.yaml
```
and the yaml for the gp2 encrypted volumes is:
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: gp2-encrypted
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  encrypted: "true"
  zones: eu-west-2a,eu-west-2b,eu-west-2c
allowVolumeExpansion: true
```


## Set up a NGINX Ingress Controller for your web services

To install a **NGINX Ingress Controller** you can once again use `helm install` with the specific **NGINX** yaml values for the Kubernetes cluster deployed. The yaml is adopted from the official repos and allows for two controller replicas.

```
helm install stable/nginx-ingress --name nginx-ingress -f nginx-ingress-values.yaml
```

## Set up Prometheus and Grafana for monitoring

Finally you can use `helm install` to deploy Prometheus and Grafana with values that have been applied to the official yaml files available.

First install Prometheus:
```
helm install stable/prometheus --name prometheus -f prometheus-values.yaml
```
and then Grafana:
```
helm install stable/grafana --name grafana -f grafana-values.yaml