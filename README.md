# Kubernetes cluster setup with minikube then AWS EKS

## Goal

[todo]

```
                                                                                                       
                               .───────────────────────────────────────────.                           
                              (                  internet                   )                          
                               `────────────────────┬──────────────────────'                           
                                                    │                                                  
                                                    │                                                  
                                                    ▼                                                  
  ┌───────────────────────────────────────────────────────────────────────────────────────────────────┐
  │        load balancer handling incoming traffic to nginx exposed via Load Balancer service         │
  └───────────────────────────────────────────────────────────────────────────────────────────────────┘
  ┌───────────────────────────────────────────────────────────────────────────────────────────────────┐
  │                                 Kubernetes simple-webapp cluster                                  │
  │                                                                                                   │
  │ ┌───────────────────────────────────────────────────────────────────────────────────────────────┐ │
  │ │                                                                                               │ │
  │ │          Kubernetes control plane (apiserver, etcd, scheduler, controller-managers)           │ │
  │ │                                                                                               │ │
  │ └───────────────────────────────────────────────────────────────────────────────────────────────┘ │
  │ ┌─────────────────────────────────────────────┐   ┌─────────────────────────────────────────────┐ │
  │ │                   node 1                    │   │                   node 2                    │ │
  │ │                                             │   │                                             │ │
  │ │   ┌─────────────────────────────────────┐   │   │   ┌─────────────────────────────────────┐   │ │
  │ │   │    Kubernetes kubelet, container    │   │   │   │    Kubernetes kubelet, container    │   │ │
  │ │   │       runtime, proxies, etc.        │   │   │   │       runtime, proxies, etc.        │   │ │
  │ │   └─────────────────────────────────────┘   │   │   └─────────────────────────────────────┘   │ │
  │ │                                             │   │                                             │ │
  │ │ ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┼ ─ ┼ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐ │ │
  │ │                 nginx deployment exposed externally via LoadBalancer service                  │ │
  │ │ │ ┌─────────────────┐ ┌─────────────────┐   │   │   ┌─────────────────┐ ┌─────────────────┐ │ │ │
  │ │   │      pod 1      │ │      pod 2      │   │   │   │      pod 3      │ │      pod 4      │   │ │
  │ │ │ │ ┌─────────────┐ │ │ ┌─────────────┐ │   │   │   │ ┌─────────────┐ │ │ ┌─────────────┐ │ │ │ │
  │ │   │ │    nginx    │ │ │ │    nginx    │ │   │   │   │ │    nginx    │ │ │ │    nginx    │ │   │ │
  │ │ │ │ │  container  │ │ │ │  container  │ │   │   │   │ │  container  │ │ │ │  container  │ │ │ │ │
  │ │   │ └─────────────┘ │ │ └─────────────┘ │   │   │   │ └─────────────┘ │ │ └─────────────┘ │   │ │
  │ │ │ └─────────────────┘ └─────────────────┘   │   │   └─────────────────┘ └─────────────────┘ │ │ │
  │ │  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │ │
  │ │                                             │   │                                             │ │
  │ │ ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┼ ─ ┼ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐ │ │
  │ │         simple-webapp deployment, available internally to nginx via ClusterIP service         │ │
  │ │ │ ┌─────────────────┐ ┌─────────────────┐   │   │   ┌─────────────────┐ ┌─────────────────┐ │ │ │
  │ │   │      pod 1      │ │      pod 2      │   │   │   │      pod 3      │ │      pod 4      │   │ │
  │ │ │ │ ┌─────────────┐ │ │ ┌─────────────┐ │   │   │   │ ┌─────────────┐ │ │ ┌─────────────┐ │ │ │ │
  │ │   │ │simple-webapp│ │ │ │simple-webapp│ │   │   │   │ │simple-webapp│ │ │ │simple-webapp│ │   │ │
  │ │ │ │ │  container  │ │ │ │  container  │ │   │   │   │ │  container  │ │ │ │  container  │ │ │ │ │
  │ │   │ └─────────────┘ │ │ └─────────────┘ │   │   │   │ └─────────────┘ │ │ └─────────────┘ │   │ │
  │ │ │ └─────────────────┘ └─────────────────┘   │   │   └─────────────────┘ └─────────────────┘ │ │ │
  │ │  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │ │
  │ └─────────────────────────────────────────────┘   └─────────────────────────────────────────────┘ │
  │                                                                                                   │
  └───────────────────────────────────────────────────────────────────────────────────────────────────┘
```

## Requirements

* [Docker](https://www.docker.com/)
* [kubectl](https://kubernetes.io/docs/reference/kubectl/)
* [minikube](https://minikube.sigs.k8s.io/)
* [AWS CLI](https://aws.amazon.com/cli/)
* [eksctl](https://eksctl.io/)

Get Docker from [here](https://docs.docker.com/get-docker/).

To install the rest in Homebrew on macOS:

    brew install kubectl minikube awscli eksctl

## AWS CLI setup

[todo]

## Build _simple-webapp_ container image

Ensure Docker is running.

Build the _simple-webapp_ container image and push it to your own repo as per my instructions [here](https://github.com/mattbrock/simple-webapp). Alternatively, avoid this step by using my _cetre/simple-webapp_ repo on Docker Hub, though I can't necessarily guarantee this image will contain the most recent version.

## Local cluster with minikube

Ensure Docker is running.

Start minikube:

    minikube start
    
Should see the following:

    $ minikube start
	😄  minikube v1.31.2 on Darwin 13.4.1 (arm64)
	✨  Automatically selected the docker driver
	📌  Using Docker Desktop driver with root privileges
	👍  Starting control plane node minikube in cluster minikube
	🚜  Pulling base image ...
	🔥  Creating docker container (CPUs=2, Memory=4000MB) ...
	🐳  Preparing Kubernetes v1.27.4 on Docker 24.0.4 ...
	    ▪ Generating certificates and keys ...
	    ▪ Booting up control plane ...
	    ▪ Configuring RBAC rules ...
	🔗  Configuring bridge CNI (Container Networking Interface) ...
	🔎  Verifying Kubernetes components...
	    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
	🌟  Enabled addons: storage-provisioner, default-storageclass
	🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default

Verify that kubectl can access the cluster:

	$ kubectl get pod -A
	NAMESPACE     NAME                               READY   STATUS    RESTARTS       AGE
	kube-system   coredns-5d78c9869d-9b979           1/1     Running   0              6m11s
	kube-system   etcd-minikube                      1/1     Running   0              6m26s
	kube-system   kube-apiserver-minikube            1/1     Running   0              6m26s
	kube-system   kube-controller-manager-minikube   1/1     Running   0              6m27s
	kube-system   kube-proxy-zgz4z                   1/1     Running   0              6m11s
	kube-system   kube-scheduler-minikube            1/1     Running   0              6m26s
	kube-system   storage-provisioner                1/1     Running   1 (6m7s ago)   6m25s
	
Change Docker repo in these files if necessary.


Do SW deployment then service:

	$ kubectl apply -f simple-webapp-deployment.yml 
	deployment.apps/simple-webapp created
	
	$ kubectl apply -f simple-webapp-service.yml 
	service/simple-webapp-svc created

Check they're running:

	$ kubectl get deployment simple-webapp
    NAME            READY   UP-TO-DATE   AVAILABLE   AGE
	simple-webapp   3/3     3            3           3m5s
	
	$ kubectl get service simple-webapp-svc
	NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
	simple-webapp-svc   ClusterIP   10.105.121.100   <none>        8080/TCP   3m48s
	
Port forward to service to check it responds:

	$ kubectl port-forward service/simple-webapp-svc 8080:8080
	Forwarding from 127.0.0.1:8080 -> 8080
	Forwarding from [::1]:8080 -> 8080
	
Check in web browser `http://localhost:8080`. CTRL-C to stop port-forwarding.

Check logs:

	$ kubectl logs -l app=simple-webapp
	127.0.0.1 - - [22/Aug/2023 07:10:27] "GET / HTTP/1.1" 200 -
	
Do nginx config map, deployment and service then check:

	$ kubectl apply -f nginx-config.yml
	configmap/nginx-config created
	$ kubectl apply -f nginx-deployment.yml
	deployment.apps/nginx created
	$ kubectl apply -f nginx-service.yml 
	service/nginx-svc created

Check:

    $ kubectl get configmap nginx-config
    NAME           DATA   AGE
    nginx-config   1      7m29s

	$ kubectl get deployment nginx
	NAME    READY   UP-TO-DATE   AVAILABLE   AGE
	nginx   3/3     3            3           7m34s
	
	$ kubectl get service nginx-svc
	NAME        TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
	nginx-svc   LoadBalancer   10.110.39.225   <pending>     8000:32689/TCP   7m49s

LB IP shows as "pending". In a separate tab, create a tunnel to the LB:

    $ minikube tunnel
	✅  Tunnel successfully started
	
	📌  NOTE: Please do not close this terminal as this process must stay alive for the tunnel to be accessible ...
	
	🏃  Starting tunnel for service nginx-svc.
	
In main tab, check service again, it now has an external IP (localhost):

	$ kubectl get service nginx-svc
	NAME        TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
	nginx-svc   LoadBalancer   10.110.39.225   127.0.0.1     8000:32689/TCP   10m

IP now assigned. Check in web browser `http://localhost:8000` then check logs:

	$ kubectl logs -l app=nginx
	10.244.0.1 - - [22/Aug/2023:07:31:37 +0000] "GET / HTTP/1.1" 200 418 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/116.0.0.0 Safari/537.36" "-"

In a separate tab, open the minikube dashboard:

	$ minikube dashboard
	🔌  Enabling dashboard ...
	    ▪ Using image docker.io/kubernetesui/dashboard:v2.7.0
	    ▪ Using image docker.io/kubernetesui/metrics-scraper:v1.0.8
	💡  Some dashboard features require the metrics-server addon. To enable all features please run:
	
	        minikube addons enable metrics-server
	
	
	🤔  Verifying dashboard health ...
	🚀  Launching proxy ...
	🤔  Verifying proxy health ...
	🎉  Opening http://127.0.0.1:50809/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/ in your default browser...

The dashboard should open in your web browser:

![minikube dashboard](README-images/minikube-dashboard.png)

Destroy local cluster in minikube:

    minikube delete
    
## Amazon EKS cluster with eksctl

Ensure AWS CLI environment set up correctly etc.

Change region in eksctl config.

Create cluster using eksctl config file:

	$ eksctl create cluster -f eks-cluster.yml
	2023-08-22 14:54:07 [ℹ]  eksctl version 0.153.0-dev+a79b3826a.2023-08-18T10:03:46Z
	2023-08-22 14:54:07 [ℹ]  using region eu-west-2
	2023-08-22 14:54:07 [ℹ]  setting availability zones to [eu-west-2c eu-west-2a eu-west-2b]
	2023-08-22 14:54:07 [ℹ]  subnets for eu-west-2c - public:192.168.0.0/19 private:192.168.96.0/19
	2023-08-22 14:54:07 [ℹ]  subnets for eu-west-2a - public:192.168.32.0/19 private:192.168.128.0/19
	2023-08-22 14:54:07 [ℹ]  subnets for eu-west-2b - public:192.168.64.0/19 private:192.168.160.0/19
	2023-08-22 14:54:07 [ℹ]  nodegroup "simple-webapp-nodegroup" will use "ami-0f3589a95ffd274bf" [AmazonLinux2/1.27]
	2023-08-22 14:54:07 [ℹ]  using Kubernetes version 1.27
	2023-08-22 14:54:07 [ℹ]  creating EKS cluster "simple-webapp" in "eu-west-2" region with un-managed nodes
	2023-08-22 14:54:07 [ℹ]  1 nodegroup (simple-webapp-nodegroup) was included (based on the include/exclude rules)
	2023-08-22 14:54:07 [ℹ]  will create a CloudFormation stack for cluster itself and 1 nodegroup stack(s)
	2023-08-22 14:54:07 [ℹ]  will create a CloudFormation stack for cluster itself and 0 managed nodegroup stack(s)
	2023-08-22 14:54:07 [ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=eu-west-2 --cluster=simple-webapp'
	2023-08-22 14:54:07 [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "simple-webapp" in "eu-west-2"
	2023-08-22 14:54:07 [ℹ]  CloudWatch logging will not be enabled for cluster "simple-webapp" in "eu-west-2"
	2023-08-22 14:54:07 [ℹ]  you can enable it with 'eksctl utils update-cluster-logging --enable-types={SPECIFY-YOUR-LOG-TYPES-HERE (e.g. all)} --region=eu-west-2 --cluster=simple-webapp'
	2023-08-22 14:54:07 [ℹ]  
	2 sequential tasks: { create cluster control plane "simple-webapp", 
	    2 sequential sub-tasks: { 
	        wait for control plane to become ready,
	        create nodegroup "simple-webapp-nodegroup",
	    } 
	}
	2023-08-22 14:54:07 [ℹ]  building cluster stack "eksctl-simple-webapp-cluster"
	2023-08-22 14:54:07 [ℹ]  deploying stack "eksctl-simple-webapp-cluster"
	2023-08-22 14:54:37 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-cluster"
	2023-08-22 14:55:08 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-cluster"
	2023-08-22 14:56:08 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-cluster"
	2023-08-22 14:57:08 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-cluster"
	2023-08-22 14:58:08 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-cluster"
	2023-08-22 14:59:08 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-cluster"
	2023-08-22 15:00:08 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-cluster"
	2023-08-22 15:01:08 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-cluster"
	2023-08-22 15:02:08 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-cluster"
	2023-08-22 15:04:09 [ℹ]  building nodegroup stack "eksctl-simple-webapp-nodegroup-simple-webapp-nodegroup"
	2023-08-22 15:04:09 [ℹ]  --nodes-min=2 was set automatically for nodegroup simple-webapp-nodegroup
	2023-08-22 15:04:09 [ℹ]  --nodes-max=2 was set automatically for nodegroup simple-webapp-nodegroup
	2023-08-22 15:04:10 [ℹ]  deploying stack "eksctl-simple-webapp-nodegroup-simple-webapp-nodegroup"
	2023-08-22 15:04:10 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-nodegroup-simple-webapp-nodegroup"
	2023-08-22 15:04:40 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-nodegroup-simple-webapp-nodegroup"
	2023-08-22 15:05:18 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-nodegroup-simple-webapp-nodegroup"
	2023-08-22 15:06:14 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-nodegroup-simple-webapp-nodegroup"
	2023-08-22 15:07:23 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-nodegroup-simple-webapp-nodegroup"
	2023-08-22 15:07:23 [ℹ]  waiting for the control plane to become ready
	2023-08-22 15:07:23 [✔]  saved kubeconfig as "/Users/brock/.kube/config"
	2023-08-22 15:07:23 [ℹ]  no tasks
	2023-08-22 15:07:23 [✔]  all EKS cluster resources for "simple-webapp" have been created
	2023-08-22 15:07:23 [ℹ]  adding identity "arn:aws:iam::318951153566:role/eksctl-simple-webapp-nodegroup-si-NodeInstanceRole-WKJIY951Q1WZ" to auth ConfigMap
	2023-08-22 15:07:23 [ℹ]  nodegroup "simple-webapp-nodegroup" has 0 node(s)
	2023-08-22 15:07:23 [ℹ]  waiting for at least 2 node(s) to become ready in "simple-webapp-nodegroup"
	2023-08-22 15:07:51 [ℹ]  nodegroup "simple-webapp-nodegroup" has 2 node(s)
	2023-08-22 15:07:51 [ℹ]  node "ip-192-168-23-36.eu-west-2.compute.internal" is ready
	2023-08-22 15:07:51 [ℹ]  node "ip-192-168-82-170.eu-west-2.compute.internal" is ready
	2023-08-22 15:07:51 [✖]  parsing kubectl version string  (upstream error: ) / "0.0.0": Version string empty
	2023-08-22 15:07:51 [ℹ]  cluster should be functional despite missing (or misconfigured) client binaries
	2023-08-22 15:07:51 [✔]  EKS cluster "simple-webapp" in "eu-west-2" region is ready
	
These lines seem to suggest a problem in setting up the config for kubectl, but I didn't experience any problems using kubectl with the EKS cluster:

	2023-08-22 15:07:51 [✖]  parsing kubectl version string  (upstream error: ) / "0.0.0": Version string empty
	2023-08-22 15:07:51 [ℹ]  cluster should be functional despite missing (or misconfigured) client binaries

Verify that kubectl can access the cluster:

	$ kubectl get pods -A
	NAMESPACE     NAME                      READY   STATUS    RESTARTS   AGE
	kube-system   aws-node-9kh9c            1/1     Running   0          6m37s
	kube-system   aws-node-nxpjf            1/1     Running   0          6m33s
	kube-system   coredns-8b48879c8-xjp9k   1/1     Running   0          14m
	kube-system   coredns-8b48879c8-zxmml   1/1     Running   0          14m
	kube-system   kube-proxy-rqscq          1/1     Running   0          6m37s
	kube-system   kube-proxy-w5wbk          1/1     Running   0          6m33s

Change Docker repo in these files if necessary.

Do SW deployment then service:

	$ kubectl apply -f simple-webapp-deployment.yml 
	deployment.apps/simple-webapp created
	
	$ kubectl apply -f simple-webapp-service.yml 
	service/simple-webapp-svc created

Check they're running:

	$ kubectl get deployment simple-webapp
	NAME            READY   UP-TO-DATE   AVAILABLE   AGE
	simple-webapp   3/3     3            3           46s
	
	$ kubectl get service simple-webapp-svc
	NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
	simple-webapp-svc   ClusterIP   10.100.64.16   <none>        8080/TCP   57s
		
Do nginx config map, deployment and service then check:

	$ kubectl apply -f nginx-config.yml
	configmap/nginx-config created
	$ kubectl apply -f nginx-deployment.yml
	deployment.apps/nginx created
	$ kubectl apply -f nginx-service.yml 
	service/nginx-svc created

Check:

    $ kubectl get configmap nginx-config
    NAME           DATA   AGE
    nginx-config   1      80s

	$ kubectl get deployment nginx
	NAME    READY   UP-TO-DATE   AVAILABLE   AGE
	nginx   3/3     3            3           91s
	
	$ kubectl get service nginx-svc
	NAME        TYPE           CLUSTER-IP      EXTERNAL-IP                                                              PORT(S)          AGE
	nginx-svc   LoadBalancer   10.100.51.154   a7ddfba7461144ebbb6ad2be87dc9127-897879486.eu-west-2.elb.amazonaws.com   8000:31815/TCP   96s
	
Using the external IP obtained above, it should now be possible to check the web app in a browser using the following URL, changing the domain to the one you've received from EKS, but retaining port 8000 at the end:

`http://a7ddfba7461144ebbb6ad2be87dc9127-897879486.eu-west-2.elb.amazonaws.com:8000`

Check the front-end (nginx deployment) and back-end (simple-webapp deployment) logs:

    $ kubectl logs -l app=nginx
    192.168.82.170 - - [22/Aug/2023:14:28:28 +0000] "GET / HTTP/1.1" 200 418 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/116.0.0.0 Safari/537.36" "-"
    
    $ kubectl logs -l app=simple-webapp
    192.168.79.249 - - [22/Aug/2023 14:28:28] "GET / HTTP/1.0" 200 -

The web-based EKS Management Console isn't as pretty as the minikube dashboard, and oddly it seems to have some missing information (I couldn't find the nginx ConfigMap in there, for example) but is still useful for obtaining information. For example, we can see our Deployments, Services and Nodes:

![EKS Console - Deployments](README-images/eks-console-deployments.png)

![EKS Console - Services](README-images/eks-console-services.png)

![EKS Console - Nodes](README-images/eks-console-nodes.png)

Destroy cluster (include waiting for CloudFormation stack to finish):

	$ eksctl delete cluster -f eks-cluster.yml --wait
	2023-08-22 14:45:20 [ℹ]  deleting EKS cluster "simple-webapp"
	2023-08-22 14:45:21 [ℹ]  will drain 1 unmanaged nodegroup(s) in cluster "simple-webapp"
	2023-08-22 14:45:21 [ℹ]  starting parallel draining, max in-flight of 1
	2023-08-22 14:45:21 [ℹ]  cordon node "ip-192-168-30-190.eu-west-2.compute.internal"
	2023-08-22 14:45:21 [ℹ]  cordon node "ip-192-168-78-162.eu-west-2.compute.internal"
	2023-08-22 14:46:43 [✔]  drained all nodes: [ip-192-168-30-190.eu-west-2.compute.internal ip-192-168-78-162.eu-west-2.compute.internal]
	2023-08-22 14:46:43 [ℹ]  deleted 0 Fargate profile(s)
	2023-08-22 14:46:44 [✔]  kubeconfig has been updated
	2023-08-22 14:46:44 [ℹ]  cleaning up AWS load balancers created by Kubernetes objects of Kind Service or Ingress
	2023-08-22 14:47:09 [ℹ]  
	2 sequential tasks: { delete nodegroup "simple-webapp-nodegroup", delete cluster control plane "simple-webapp" 
	}
	2023-08-22 14:47:10 [ℹ]  will delete stack "eksctl-simple-webapp-nodegroup-simple-webapp-nodegroup"
	2023-08-22 14:47:10 [ℹ]  waiting for stack "eksctl-simple-webapp-nodegroup-simple-webapp-nodegroup" to get deleted
	2023-08-22 14:47:10 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-nodegroup-simple-webapp-nodegroup"
	2023-08-22 14:47:40 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-nodegroup-simple-webapp-nodegroup"
	2023-08-22 14:48:38 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-nodegroup-simple-webapp-nodegroup"
	2023-08-22 14:49:21 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-nodegroup-simple-webapp-nodegroup"
	2023-08-22 14:49:22 [ℹ]  will delete stack "eksctl-simple-webapp-cluster"
	2023-08-22 14:49:22 [ℹ]  waiting for stack "eksctl-simple-webapp-cluster" to get deleted
	2023-08-22 14:49:22 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-cluster"
	2023-08-22 14:49:52 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-cluster"
	2023-08-22 14:50:33 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-cluster"
	2023-08-22 14:52:14 [ℹ]  waiting for CloudFormation stack "eksctl-simple-webapp-cluster"
	2023-08-22 14:52:14 [✔]  all cluster resources were deleted

When wishing to administer the cluster from a different workstation, to update the minikube config for the EKS cluster:

    eksctl utils write-kubeconfig -c simple-webapp