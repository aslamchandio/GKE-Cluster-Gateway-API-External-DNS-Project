# GKE Standard Semi Private Regional Clusters


## How cluster network isolation works

In a GKE cluster, network isolation depends on who can access the cluster components and how. You can control:

Control plane access: You can customize external access, limited access, or unrestricted access to the control plane.
Cluster networking: You can choose who can access the nodes in Standard clusters, or the workloads in Autopilot clusters.
Before you create your cluster, consider the following:

Who can access the control plane and how is the control plane exposed?
How are your nodes or workloads exposed?
To answer these questions, follow the plan and design guidelines in About network isolation.

This document shows how to create a Standard regional cluster to increase availability of the cluster's control plane and workloads during cluster upgrades, automated maintenance, or a zonal disruption.


## Control plane

You can add up to 50 authorized networks (allowed CIDR blocks) in a project . For more information, refer to Define the IP addresses that can access the control plane.

While GKE can detect overlap with the control plane address block, it cannot detect overlap within a Shared VPC network.

## Cluster networking

Internal IP addresses for nodes come from the primary IP address range of the subnet you choose for the cluster. Pod IP addresses and Service IP addresses come from two subnet secondary IP address ranges of that same subnet. For more information, see IP ranges for VPC-native clusters.

GKE supports any internal IP address ranges, including private ranges (RFC 1918 and other private ranges) and privately used external IP address ranges. See the VPC documentation for a list of valid internal IP address ranges.

If you expand the primary IP range of a subnet to accommodate additional nodes, then you must add the expanded subnet's primary IP address range to the list of authorized networks for your cluster. If you don't, ingress-allow firewall rules relevant to the control plane aren't updated, and new nodes created in the expanded IP address space won't be able to register with the control plane. This can lead to an outage where new nodes are continuously deleted and replaced. Such an outage can happen when performing node pool upgrades or when nodes are automatically replaced due to liveness probe failures.

All nodes in a cluster with only private nodes are created without an external IP; they have limited access to Google Cloud APIs and services. To provide outbound internet access for your private nodes, you can use Cloud NAT.

Private Google Access is enabled automatically when you create a cluster unless you are using Shared VPC. You must not disable Private Google Access unless you are using NAT to access the internet.

### Semi Private Cluster 

```
gcloud config set accessibility/screen_reader false
gcloud config set compute/region us-central
gcloud config set compute/zone us-central1-c

```

```

gcloud container clusters create private-cluster1 \
    --region us-central1 \
    --tier standard \
    --labels=env=prod-cluster,team=it \
    --node-locations us-central1-c,us-central1-f \
    --release-channel "regular" \
    --cluster-version 1.31.6-gke.1064000 \
    --num-nodes 1 \
    --enable-master-authorized-networks \
    --master-authorized-networks 172.22.4.15/32,39.51.108.78/32 \
    --enable-authorized-networks-on-private-endpoint \
    --private-endpoint-subnetwork k8s-master-sub1-us-central1 \
    --network k8s-vpc \
    --subnetwork k8s-sub1-us-central1  \
    --cluster-secondary-range-name pod-cidr1 \
    --services-secondary-range-name service-cidr \
    --enable-private-nodes \
    --enable-ip-alias \
    --enable-master-global-access \
    --enable-dns-access \
    --enable-google-cloud-access \
    --gateway-api standard \
    --machine-type e2-medium \
    --enable-shielded-nodes \
    --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver,GcpFilestoreCsiDriver,GcsFuseCsiDriver \
    --disk-type pd-balanced  \
    --disk-size 30 \
    --default-max-pods-per-node 110 \
    --service-account gke-sa-np@dev-project-450808.iam.gserviceaccount.com \
    --scopes=https://www.googleapis.com/auth/cloud-platform \
    --enable-dataplane-v2 \
    --enable-dataplane-v2-metrics  \
    --enable-dataplane-v2-flow-observability \
    --enable-autoupgrade \
    --enable-autorepair \
    --max-surge-upgrade 1 \
    --max-unavailable-upgrade 0 \
    --enable-managed-prometheus \
    --enable-shielded-nodes \
    --no-enable-basic-auth \
    --workload-pool dev-project-786111.svc.id.goog \
    --no-issue-client-certificate


```

1- CLUSTER_NAME: the name of your new regional cluster.
2- COMPUTE_REGION: the region for your cluster, such as us-central1.
3- COMPUTE_ZONE: the zone for your node pool, such as us-central1-a. The zone must be in the same region as the cluster control plane.
4- CHANNEL: the type of release channel, which can be one of rapid, regular, stable, or None. By default, the cluster is enrolled in the regular release channel unless at least one of the following flags is specified: --cluster-version, --release-channel, --no-enable-autoupgrade, and --no-enable-autorepair.
VERSION: the version you want to specify for your cluster.

You can configure the IP addresses that can access the control plane external and internal endpoints by using the following flags:

enable-master-authorized-networks: Specifies that access to the external endpoint is restricted to IP address ranges that you authorize.

master-authorized-networks: Lists the CIDR values for the authorized networks. This list is comma-delimited list. For example, 8.8.8.8/32,8.8.8.0/24.

enable-authorized-networks-on-private-endpoint: Specifies that access to the internal endpoint is restricted to IP address ranges that you authorize with the enable-master-authorized-networks flag.

no-enable-google-cloud-access: Denies access to the control plane from Google Cloud external IP addresses.

enable-master-global-access: Allows access from IP addresses in other Google Cloud regions.

#### Get Cluster Detail

```

gcloud container clusters list
gcloud container clusters describe private-cluster1 --region us-central1

```

#### Deescribe Cluster Config File

```

gcloud container clusters describe private-cluster1 \
   --location=us-central1 \
   --format="yaml(network, privateClusterConfig)"

```   

#### Check master-authorized-networks 

```

gcloud container clusters describe private-cluster1 --format "flattened(masterAuthorizedNetworksConfig.cidrBlocks[])" --region us-central1  

```

#### Add master-authorized-networks  IPS

```
gcloud container clusters update private-cluster1 \
    --region us-central1 \
    --enable-master-authorized-networks \
    --master-authorized-networks 39.45.110.82/32,172.21.5.0/24

gcloud container clusters describe private-cluster1 --format "flattened(masterAuthorizedNetworksConfig.cidrBlocks[])" --region us-central1      

```

#### Confirm the Gateway API is enabled in the GKE control plane

```
gcloud container clusters describe private-cluster1 \
  --region us-central1   \
  --format json     

```

#### Enable Autoscalling for Cluster

```
gcloud container node-pools list --cluster private-cluster1 --region us-central1

gcloud container node-pools describe default-pool --cluster private-cluster1 --region us-central1

gcloud container clusters update private-cluster1 \
    --enable-autoscaling \
    --region us-central1 \
    --node-pool  default-pool \
    --min-nodes 1  \
    --max-nodes 2 \
    --location-policy BALANCED    

```

#### Connect GKE Cluster

```

gcloud container clusters list

gcloud container clusters get-credentials private-cluster1 --region us-central1 --project dev-project-786111 --internal-ip     

```


### Delete Clusters

```
gcloud container clusters list

gcloud container clusters delete private-cluster1 --region us-central1

```

### NatGW for Private Nodes in GKE:

#### Cloud NAT for Region US-Central1

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

