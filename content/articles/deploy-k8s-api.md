---
author: "Dimitris Traskas"
date: 2020-08-10
linktitle: Deploy Kubernetes and a Restful API on AWS in Just 20 Minutes 
menu:
  main:
    parent: 
next: 
prev: 
title: Deploy Kubernetes and a Restful API on AWS in Just 20 Minutes
weight: 10
summary: "In the time it takes to brew some fresh coffee, you will have a Kubernetes cluster and a simple Todo API, up and running."
BookToC: true
---

*Originally published in [Medium.](https://medium.com/swlh/deploy-kubernetes-and-a-restful-api-on-aws-in-just-20-minutes-353372da6216?source=friends_link&sk=2b4e5928d885d1fbc12819ef7a45633d)*

Just a few years back, setting up Kubernetes clusters and deploying Microservices and APIs used to take an awful lot of time. Initially there were no managed Kubernetes clusters and everyone had to roll up their sleeves and set up something from scratch. The introduction of managed clusters by **AWS, Google Cloud, and Azure** was a game-changer, and made Kubernetes much more accessible.

What I will try and demonstrate here, is how much faster it has become to provision Kubernetes and APIs these days. You still need to have of course in-depth knowledge and understand what you are doing. You still need to have decent skills with the command line and experience with a variety of languages and AWS. But tools have improved and something that used to take hours can now take a few minutes.

In the next few sections, I will share with you the necessary tools and a step by step guide to provision a Kubernetes cluster, deploy a simple Todo API written in Python and expose it via an AWS Elastic Load Balancer. This entire process usually takes me around 20 minutes, enough time to go and get some fresh coffee.

So let’s dive straight in!

---

## Getting started
I will admit that you need to have a few scripts and repositories ready before you try this. Links to the repos are made available in later sections. You will also need to have decent skills with the command line. I am using zsh as my default shell, with decent autocompletion that really boosts my productivity here.

Before we even start though, let’s check the list of prerequisites, from tools to install, to AWS services you need to be familiar with:

-   [**AWS EKS**](https://aws.amazon.com/eks/): stands for Elastic Kubernetes Service and offers managed Kubernetes clusters. The managed part, means that Amazon provides scalable and highly-available master nodes deployed across multiple AWS availability zones
-   [**AWS Cloudformation**](https://aws.amazon.com/cloudformation/): used to provision AWS resources such as servers, Virtual Private Networks (VPC), service accounts and more
-   [**AWS ELB**](https://aws.amazon.com/elasticloadbalancing/): is a load balancer that distributes incoming traffic to our cluster services via an NGINX Ingress Controller
-   [**NGINX Ingress Controller**](https://docs.nginx.com/nginx-ingress-controller/): is an application that runs in the cluster and configures an HTTP load balancer according to Ingress resources. In this demo we will use it to expose our API via an AWS ELB
-   [**AWS CLI**](https://aws.amazon.com/cli/): the command line tool you really need to know if you want to provision or manage resources on AWS
-   [**AWS IAM**](https://aws.amazon.com/iam/): this is where access to resources is managed
-   [**AWS ECR**](https://aws.amazon.com/ecr/): stands for Elastic Container Registry and it’s the Amazon Docker container registry where we will push our Todo API
-   [**eksctl**](https://eksctl.io/): this is the CLI tool that we are going to use to deploy the cluster. Effectively `eksctl` creates `Cloudformation` stacks and offers the YAML approach in parameterising and provisioning Kubernetes
-   [**kubectl**](https://kubernetes.io/docs/tasks/tools/install-kubectl/): the Kubernetes CLI tool
-   [**Helm**](https://helm.sh/): this is the package manager for Kubernetes that we’ll be using to install applications in our cluster

## Timer starts at 00:00 — Provision Kubernetes
Before we start our imaginary timer and provision a new cluster, you will need to download the tools mentioned earlier and more specifically eksctl. Initially developed by Weaveworks and later adopted by AWS, `eksctl` covers a range of commands for creating, managing and deleting clusters. It allows you to create groups of nodes for different workloads, set up service accounts and more.

Once you download the tool you can create a new cluster using the `create cluster` command and a configuration `YAML` file like the one in Figure 1. This file contains a few parameters such as the number and type of nodes, networking configuration, access control to AWS services, etc.

Before you start, you have to really decide what kind of cluster you need. This is a simple demo, so I am provisioning a small cluster with 3 worker nodes of `m5.large` EC2 type on three availability zones. One thing to note, is that you will have to replace the serviceRoleARN with the one you set up on **AWS IAM**.

```yaml
# ClusterConfig object creating a new VPC and 3 workers:
--- 
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: dev1
  region: eu-west-2

iam:
  serviceRoleARN: "<Your eksServiceRole has to be replaced with yours>"
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
  - name: web-workers
    instanceType: m5.large
    minSize: 1
    maxSize: 3
    desiredCapacity: 3
    amiFamily: AmazonLinux2
    ami: auto
    volumeSize: 20
    availabilityZones: ["eu-west-2a", "eu-west-2b", "eu-west-2c"]
    labels:
      nodegroup-type: frontend-workloads
    privateNetworking: true # if only 'Private' subnets are given, this must be enabled
    ssh: # use existing EC2 key but don't allow SSH access to nodegroup.
      publicKeyName: ec2_dev_key      
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
      k8s.io/cluster-autoscaler/dev1: "owned"
```

Once your `cluster.yaml` is ready you can execute the command below and start the timer.

```bash
eksctl create cluster -f cluster.yaml
```

It usually takes around **15 minutes** in my region to provision a new cluster and as you can see below, there is plenty of logging information.

![alt-text](/../../photos/cluster.png)

After 15 minutes, once the cluster is up and running, we install the Kubernetes dashboard, with three successive commands using a config file for the Metrics server, Kubernetes dashboard and the required EKS service account. The `YAML` files referenced below and the config for the cluster can be found in the Github repo here.

```bash
kubectl apply -f metrics-components.yaml
kubectl apply -f kubernetes-dashboard.yaml
kubectl apply -f eks-admin-service-account.yaml
```

Once the dashboard is installed, you can access it by running kubectl proxy and visiting the page below:
```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#!/login
```

You will be asked for an authentication token which you can retrieve by executing:

```bash
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
```

And viola! The dashboard shows 3 healthy nodes up and running, in three availability zones and with minimal CPU and Memory utilization.

![alt-text](/../../photos/dashboard.png)

## Timer at 00:16 - Set up the Todo API

After 16 minutes, we have only a tiny bit of time left to deploy our demo API. There are 1000s of todo APIs available on the web but for the purposes of this article, I developed something really simple with a few endpoints available. The API is written in Python and the Flask framework, and uses Helm charts for deployment. The source code can be found here and has some basic instructions on how to build and push the Docker images to AWS ECR.

The API is serving to clients via three endpoints, one used by Kubernetes to check the health of the service, and two more to add or retrieve todo items.

```bash
GET /healthz
POST /tasks
GET /tasks
```

To use this API in Kubernetes you need to first build and push a Docker image to **AWS ECR**. You create a new repository as can be seen below with security scans enabled by default. It is useful to see if there are any critical vulnerabilities in the images used by Kubernetes.

![alt-text](/../../photos/docker_repo.png)

Once the Docker image is pushed we can reference it in the deployment Helm charts with the correct tag. I often reference the `latest` tag for the image, if I am not interested in a specific version.

## Timer at 00:18 — Set up an NGINX Ingress Controller
To expose any Restful API running inside Kubernetes you will need to set up an `NGINX Ingress Controller`. Ingress Controllers are connected to an AWS Elastic Load Balancer and distribute incoming traffic to Kubernetes pods and services.

You can install an Ingress Controller using the official helm chart which resides in the repo https://kubernetes-charts.storage.googleapis.com/ . You will need to add the Helm repo first before you try and install anything. You will also need to configure NGINX so that you can expose your API via an Elastic Load Balancer.

The `nginx-ingress-values.yaml` that you can find in the repo I mentioned earlier, contains all the configuration settings you need to expose the Todo API. It is worth mentioning here, that I am using a Network Load Balancer for this set up, which allows the distribution of traffic based on network variables, such as IP address and destination ports.
```
service.beta.kubernetes.io/aws-load-balancer-type: nlb
```
Once the configuration settings are ready you can begin the installation of the NGINX Ingress Controller:

```bash
$ helm repo add stable https://kubernetes-charts.storage.googleapis.com/
$ helm repo update
$ helm install nginx-ingress stable/nginx-ingress -f nginx-ingress-values.yaml 
-n kube-system
```

It takes a few seconds and the **Elastic Load Balancer** appears ready on the AWS Console, while pods with a single Ingress Controller and a default backend are running on Kubernetes.

## Timer at 00:19 — Deploy the API
As mentioned earlier, Helm is a package manager that speeds up the deployment of applications in Kubernetes. Deployments can consist of a number of components such as replica sets, Ingress services, Cron Jobs, secrets, etc. With Helm we accelerate the deployment process and use a single file of parameter values to provision our Kubernetes resources.

For the deployment of the Todo API I am using 3 replicas that will get deployed on 3 availability zones, tiny amount of memory resources and a single Gunicorn worker. Gunicorn is a production grade web server, that is commonly used to deploy Restful APIs written in Python and Flask.

The parameter that needs attention of course is the Ingress service host, which will have to be replaced with the **internet-facing** load balancer of your AWS configuration. This value will essentially expose your API and make it accessible to the path specified, in this case `/todoapp/api/v1`.

```yaml
replicaCount: 3

image:
  tag: "latest"
  pullPolicy: "Always"
    
api:
  resources:
    requests:
      memory: 128Mi
    limits:
      memory: 256Mi
  env:
    # Number of desired Gunicorn workers
    GUNICORN_WORKERS: "1"      

ingress:
  enabled: true
  path: /todoapp
  hosts:
    - localhost
    # TODO to be replaced with your Elastic Load Balancer
    - <replace with your ELB>
```

Once all the values are ready you can execute the command below and after a few seconds have the Todo API up and running.

```bash
helm install todoapp deployment/todoapp/ -f deployment/prod-values.yaml
```

## Timer at 00:20!
At last, after 20 minutes the API is deployed with three pods running in Kubernetes (Figure 6) and exposed via the load balancer as seen in the Swagger documentation (Figure 7).

![alt-text](/../../photos/todoapps.png)

You can now create some new tasks and store them in memory or retrieve them from the GET endpoint.

![alt-text](/../../photos/todoapi.png)

From this point onwards, you can continue by setting up the **AWS API Gateway** and **AWS Cloudfront** to expose the API to the outside world securely. You can set up users in **AWS Cognito** and enable **OAuth2.0** authentication, add rate limiting and security rules in **AWS WAF**.

There is still plenty of work to do before you have a production grade, secure API, however we have only spent 20 minutes so far to create a cluster and deploy our application.

Enjoy your coffee!