# GCP VPC For Google Kubernetes Engine


## What this does?

This document describes how to create, modify, and delete Virtual Private Cloud (VPC) networks and subnetworks. Before reading this document, ensure that you're familiar with the characteristics of VPC networks as described in VPC networks. Networks and subnets are different resources in Google Cloud.

## Create networks
You can choose to create an auto mode or custom mode VPC network. Each new network that you create must have a unique name within the same project.

## Create an auto mode VPC network
When you create an auto mode VPC network, one subnet is created in each Google Cloud region. As new regions become available, new subnets in those regions are automatically added to the auto mode VPC network. IPv4 ranges for the automatically created subnets come from a predetermined set of ranges. All auto mode VPC networks use the same set of IPv4 ranges.

Subnets with IPv6 ranges are not supported on auto mode VPC networks. Create a custom mode VPC network if you want to create dual-stack subnets.

## Create a custom mode VPC network with only IPv4 subnets

For custom mode VPC networks, create a network, then create the subnets that you want within a region. You do not have to specify subnets for all regions right away, or even at all, but you cannot create instances in a region that has no subnet defined. Finally, define the firewall rules for your network.

To create a custom mode VPC network with only IPv4 subnets, follow these steps.

```
gcloud config set accessibility/screen_reader false
gcloud config set compute/region us-central
gcloud config set compute/zone us-central1-c

```

```
gcloud compute networks create NETWORK \
    --subnet-mode=custom \
    --bgp-routing-mode=DYNAMIC_ROUTING_MODE \
    --mtu=MTU

gcloud compute networks create k8s-vpc \
    --subnet-mode custom \
    --bgp-routing-mode regional \
    --mtu 1460  

# note : or --bgp-routing-mode global

gcloud compute networks list  

```

1- NETWORK: a name for the VPC network.
2- DYNAMIC_ROUTING_MODE: controls the behavior of Cloud Routers in the   network. Can be either global or regional. The default is regional. For more information, see dynamic routing mode.
3- MTU: the maximum transmission unit (MTU), which is the largest packet size of the network. MTU can be set to any value from 1300 to 8896. The default is 1460. Before setting the MTU to a value higher than 1460, review Maximum transmission unit.

Next, add subnets to your network.

Create a custom mode VPC network with a IPv4 subnet

```
gcloud compute networks subnets create k8s-sub1-us-central1 \
  --network k8s-vpc \
  --range 192.168.16.0/20 \
  --region us-central1 \
  --enable-flow-logs \
  --enable-private-ip-google-access

gcloud compute networks subnets list --network k8s-vpc
gcloud compute networks subnets list --filter  network:k8s-vpc  

```

### Add Pod & Service CIDR in Subnet

### Note: For Pod CIDR(10.244.0.0/16,10.245.0.0/16)  For Service CIDR 10.32.0.0/16

```
gcloud compute networks subnets update k8s-sub1-us-central1 \
  --region us-central1 \
  --add-secondary-ranges=pod-cidr1=10.244.0.0/16

gcloud compute networks subnets update k8s-sub1-us-central1 \
  --region us-central1 \
  --add-secondary-ranges=pod-cidr2=10.245.0.0/16 


gcloud compute networks subnets update k8s-sub1-us-central1 \
  --region us-central1 \
  --add-secondary-ranges=service-cidr=10.32.0.0/16

gcloud compute networks subnets describe k8s-sub1-us-central1 

gcloud compute networks subnets list --network k8s-vpc
gcloud compute networks subnets list --filter  network:k8s-vpc

```

Note : You can also Configure subnet with Pod and service CIDR in one command

```
gcloud compute networks subnets create k8s-sub1-us-central1 \
    --network k8s-vpc \
    --range 192.168.16.0/20 \
    --region us-central1 \    
    --secondary-range pod-cidr1=10.244.0.0/16,pod-cidr2=10.245.0.0/16,service-cidr=10.32.0.0/16 \
    --enable-flow-logs \
    --enable-private-ip-google-access
    
gcloud compute routes list \
    --filter="default-internet-gateway k8s-vpc"

```

###  Subnet For Control PLan 

```
gcloud compute networks subnets create k8s-master-sub1-us-central1 \
  --network k8s-vpc \
  --range 172.16.1.0/28 \
  --region us-central1 \
  --enable-flow-logs 

gcloud compute networks subnets list --network k8s-vpc
gcloud compute networks subnets list --filter  network:k8s-vpc

```

###  Subnet For Regional Load Balancer(Gateway API)

```
gcloud compute networks subnets create k8s-proxy-only-subnet1 \
    --network k8s-vpc \
    --range 192.168.1.0/24 \
    --region us-central1 \
    --purpose REGIONAL_MANAGED_PROXY \
    --role ACTIVE

gcloud compute networks subnets list --filter  network:k8s-vpc

```

##  Other Subnets

```
gcloud compute networks subnets create k8s-sub2-us-central1 \
  --network k8s-vpc \
  --range 172.22.1.0/24 \
  --region us-central1 \
  --enable-flow-logs \
  --enable-private-ip-google-access

gcloud compute networks subnets create k8s-sub3-us-west1 \
  --network k8s-vpc \
  --range 172.22.2.0/24 \
  --region us-west1 \
  --enable-flow-logs \
  --enable-private-ip-google-access
 

gcloud compute networks subnets create k8s-sub4-euro-west2 \
  --network k8s-vpc \
  --range 172.22.3.0/24 \
  --region europe-west2 \
  --enable-flow-logs \
  --enable-private-ip-google-access 

gcloud compute networks subnets create k8s-sub5-me-central1 \
  --network k8s-vpc \
  --range 172.22.4.0/24 \
  --region me-central1 \
  --enable-flow-logs \
  --enable-private-ip-google-access 

```

## Firewalls Rules for VPC

### SSH Allow Firewall Rule

```
gcloud compute firewall-rules create k8s-fw-ssh-allow \
    --network k8s-vpc \
    --action allow \
    --direction ingress \
    --rules tcp:22,icmp \
    --source-ranges 39.57.176.9/32 \
    --priority 1000 \
    --enable-logging \
    --target-tags k8s-ssh-allow

gcloud compute firewall-rules list --filter network:k8s-vpc
gcloud compute firewall-rules describe k8s-vpc-ssh-allow

```

### RDP Allow Firewall Rule

```
gcloud compute firewall-rules create k8s-fw-rdp-allow \
    --network k8s-vpc \
    --action allow \
    --direction ingress \
    --rules tcp:3389,icmp \
    --source-ranges 39.57.176.9/32 \
    --priority 1100 \
    --enable-logging \
    --target-tags k8s-rdp-allow

gcloud compute firewall-rules list --filter network:k8s-vpc
gcloud compute firewall-rules describe k8s-vpc-rdp-allow    

```

### Internal ALLow for Subnets Firewall Rule

```
gcloud compute firewall-rules create k8s-fw-internal-allow \
    --network k8s-vpc \
    --action allow \
    --direction ingress \
    --rules tcp,udp,icmp,ipip \
    --source-ranges 192.168.16.0/20,172.22.1.0/24,172.22.2.0/24,172.22.3.0/24,172.22.4.0/24 \
    --enable-logging \
    --priority 1200

 OR

 gcloud compute firewall-rules create k8s-fw-internal-allow \
    --network k8s-vpc \
    --action allow \
    --direction ingress \
    --rules tcp,udp,icmp,ipip \
    --source-ranges 192.168.0.0/16,172.22.0.0/16,10.0.0.0/8 \
    --enable-logging \
    --priority 1200 

gcloud compute firewall-rules list --filter network:k8s-vpc
gcloud compute firewall-rules describe k8s-fw-internal-allow  

  
```

### IAP ALLow  Firewall Rule

```
gcloud compute firewall-rules create k8s-fw-iap-allow \
    --network k8s-vpc \
    --action allow \
    --direction ingress \
    --rules tcp:22,tcp:3389,tcp:3306 \
    --source-ranges 35.235.240.0/20 \
    --enable-logging \
    --priority 1300
  

```

### HTTP ALLow  Firewall Rule

```
gcloud compute firewall-rules create k8s-fw-http-allow \
    --network k8s-vpc \
    --action allow \
    --direction ingress \
    --rules tcp:80,tcp:443,tcp:8080,tcp:9090 \
    --source-ranges 0.0.0.0/0 \
    --enable-logging \
    --priority 1400 \
    --target-tags k8s-http-allow 

gcloud compute firewall-rules list --filter network:k8s-vpc
gcloud compute firewall-rules describe k8s-fw-http-allow   
  

```

### Health Check ALLow  Firewall Rule

```
gcloud compute firewall-rules create k8s-fw-health-check-allow \
    --network k8s-vpc \
    --action allow \
    --direction ingress \
    --rules tcp:80,tcp:443,tcp:8080 \
    --source-ranges 192.168.1.0/24  \
    --enable-logging \
    --priority 1500 

gcloud compute firewall-rules describe k8s-fw-health-check-allow   
  

```

### NodePort ALLow  Firewall Rule

```
gcloud compute firewall-rules create k8s-fw-node-port-allow \
    --network k8s-vpc \
    --action allow \
    --direction ingress \
    --rules tcp:30000-32768 \
    --source-ranges 0.0.0.0/0 \
    --enable-logging \
    --priority 1600 

gcloud compute firewall-rules describe k8s-fw-node-port-allow  
  

```

### ArgoCD ALLow  Firewall Rule

```

gcloud compute firewall-rules create k8s-fw-iap-argocd-allow \
    --network k8s-vpc \
    --action allow \
    --direction ingress \
    --rules tcp:30000-32768 \
    --source-ranges 35.235.240.0/20 \
    --priority 1600 

gcloud compute firewall-rules list --filter network:k8s-vpc   
  
```

## Cloud NAT Overview


### What this does?

Cloud NAT provides network address translation (NAT) for outbound traffic to the internet, Virtual Private Cloud (VPC) networks, on-premises networks, and other cloud provider networks.

Cloud NAT provides NAT for the following Google Cloud resources:

1- Compute Engine virtual machine (VM) instances
2- Private Google Kubernetes Engine (GKE) clusters
3- Cloud Run instances through Serverless VPC Access or Direct VPC egress
4- Cloud Run functions instances through Serverless VPC Access
5- App Engine standard environment instances through Serverless VPC Access

Cloud NAT supports address translation for established inbound response packets only. It doesn't allow unsolicited inbound connections.

### NatGW for Private Nodes in GKE:

#### Cloud NAT for Region 1

```
gcloud compute addresses create natgw-gke-pip-us-central1  \
    --region us-central1

gcloud compute addresses list

gcloud compute addresses describe natgw-gke-pip-us-central1 --region us-central1

gcloud compute routers create gke-nat-router-us-central1 \
    --network k8s-vpc \
    --region us-central1

gcloud compute routers list

Note : For All Subnet

gcloud compute routers nats create gke-natgw-us-central1 \
    --router gke-nat-router-us-central1 \
    --region us-central1 \
    --nat-external-ip-pool natgw-gke-pip-us-central1 \
    --nat-all-subnet-ip-ranges \
    --min-ports-per-vm 128 \
    --max-ports-per-vm 512 \
    --enable-logging

Note : For only one Subnet

gcloud compute routers nats create gke-natgw-us-central1 \
    --router gke-nat-router-us-central1 \
    --region us-central1 \
    --nat-external-ip-pool natgw-gke-pip-us-central1 \
    --nat-custom-subnet-ip-ranges k8s-sub1-us-central1 \
    --min-ports-per-vm 128 \
    --max-ports-per-vm 512 \
    --enable-logging


gcloud compute routers nats list --router gke-nat-router-us-central1 --region us-central1
gcloud compute routers nats describe gke-natgw-us-central1 --router gke-nat-router-us-central1 --region us-central1

```

