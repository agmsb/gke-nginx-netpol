# Using Network Policies with the Nginx Ingress Controller

_Copyright 2019 Google LLC. This software is provided as-is, without warranty or representation for any use or purpose. Your use of it is subject to your agreements with Google. This is NOT a Google official product._

This tutorial shows how to ensure that a set of pods are only accessible from the nginx ingress controller using network policies. 

## Requirements

* Google Cloud Platform project
* gcloud set up and initialized for a project with the Kubernetes Engine API enabled.
* Basic Kubernetes knowledge

## Summary

You can use GKE's network policy enforcement to control the communication between your cluster's Pods and Services. To define a network policy on GKE, you can use the Kubernetes Network Policy API to create Pod-level firewall rules. These firewall rules determine which Pods and Services can access one another inside your cluster.

This repo applies network policies to ensure that only traffic from the nginx ingress controller is allowed to the pod; all other traffic from other pods will be denied. 

## Setup

Clone this repository.

Set your default compute zone and create a GKE cluster with network policy enabled.
```
gcloud config set compute/zone us-west1-a
gcloud container clusters create gke-nginx-netpol --enable-network-policy
```

Set your user to have cluster-admin clusterrole via a clusterrolebinding.
```
GCLOUD_USER=$(gcloud config get-value account)
```
```
kubectl create clusterrolebinding cluster-admin-binding \
      --clusterrole cluster-admin --user $GCLOUD_USER
```
## Deploy Sample App
Run and expose a sample application titled `hello-app`. This will create a Kubernetes deployment and a Kubernetes `ClusterIP` service.
```
kubectl run hello-app --image=gcr.io/google-samples/hello-app:1.0 --port=8080 --expose
```

## Deploy Nginx Ingress Controller and Kubernetes Ingress
Run the commands below to deploy the nginx-ingress-controller. This will create a Kubernetes deployment and a Kubernetes `LoadBalancer` service.
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
```

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/cloud-generic.yaml
```

Run the `kubectl` command below to create the Kubernetes Ingress resource that will route to our `hello-app` service.
```
kubectl apply -f ingress/ingress.yaml
```

## Create Network Policies
Run the `kubectl` command below to create the `deny-all` network policy  that will deny all incoming traffic to the default namespace.
```
kubectl apply -f netpol/deny-all.yaml
```

You then have two approaches to whitelist traffic to the `hello-app` in the default namespace.

The first is a network policy that will only allow traffic to our `hello-app` from pods with the label `app.kubernetes.io/name: ingress-nginx` and from a namespace with the label `app.kubernetes.io/name: ingress-nginx`. Apply the whitelist-ingress-nginx network policy for this.
```
kubectl apply -f netpol/whitelist-ingress-nginx.yaml
```

The second is a network policy that will whitelist traffic in the default namespace from source IPs within a given IP block. In this case, we would whitelist the entire GKE pod CIDR, meaning all pod IPs, including the ingress-nginx controller, will be able to reach our `hello-app`. 

```
gcloud container clusters describe gke-nginx-netpol --zone us-central1-a | grep clusterIpv4CidrBlock | awk '{ print $2 }'
```

Replace `spec.ingress.from.ipBlock.cidr` with this value in `netpol/whitelist-podcidr.yaml`. Then apply the whitelist-podcidr.yaml.

```
kubectl apply -f netpol/whitelist-podcidr.yaml
```

## Test Network Policy with Nginx Ingress Controller 
Run the below command and you should have your nginx ingress public IP. This typically takes a few minutes, so run the command again if the IP address is still being allocated.
```
kubectl get service ingress-nginx -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

Export the IP to `$NGINX_IP` and curl the public IP to access `hello-app` via the nginx ingress controller.
```
NGINX_IP=$(kubectl get service ingress-nginx -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}') && echo $NGINX_IP
```
```
curl http://$NGINX_IP/hello
```

Now try to access `hello-app` via a test pod running in your cluster. You will use the FQDN associated with the `hello-app` service.
```
kubectl run test-$RANDOM --rm -i -t --image=alpine -- sh
```
```
# / wget -qO- --timeout=2 http://hello-app:8080
```

You should get the below. This is our network policy in effect.
```
wget: download timed out
```
