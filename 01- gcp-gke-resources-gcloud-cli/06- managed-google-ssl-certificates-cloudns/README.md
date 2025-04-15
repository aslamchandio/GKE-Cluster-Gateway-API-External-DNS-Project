---
title: GKE with Kubernetes Gateway API - Google Managed Certificates
description: Create GCP Application Load Balancer using GKE Kubernetes Gateway API with Domain Name based routing
---

## Step-01: Introduction
1. Create GCP Application Load Balancer using Kubernetes Gateway API, Static IP for Load Balancer, Self-signed SSL with Google Cloud Certificate Manager, Domain Name based routing 
2. **Approach-1:** Using Kubernetes YAML Manifests
3. **Approach-2:** Using Terraform Manifests

### Step-02-02: Create Regional Static IP
```t
# Create Regional Load Balancer IP
gcloud compute addresses create my-regional-pip \
    --region="REGION_NAME" \
    --project=my-project-id

gcloud compute addresses create my-regional-pip \
    --region us-central1 \
    --network-tier STANDARD

# List IP Addresss    
gcloud compute addresses list
gcloud compute addresses describe my-regional-pip --region us-central1
```

### Step-02-03: Create Google  SSL certificates
```t
# Create DNS Authorization 
gcloud certificate-manager dns-authorizations create dns-authorization-cert \
    --domain cloudaxt.online \
    --type=PER_PROJECT_RECORD \
    --location us-central1 
...



### List DNS Authorization 
...  
gcloud certificate-manager dns-authorizations list

gcloud certificate-manager dns-authorizations describe dns-authorization-cert --location us-central1

data: b31744e5-87ed-4580-8107-92b2231efb5c.0.us-central1.authorize.certificatemanager.goog.
name:  _acme-challenge_6wvc2ga4536h76mg.cloudaxt.online.

...
### Start Transaction Record Sets
...

gcloud dns record-sets transaction start --zone cloudaxt-online

...

### Add Transaction Record Sets
...

CNAME_RECORD : b31744e5-87ed-4580-8107-92b2231efb5c.0.us-central1.authorize.certificatemanager.goog.

--name       :  _acme-challenge_6wvc2ga4536h76mg.cloudaxt.online.

gcloud dns record-sets transaction add CNAME_RECORD \
    --name="VALIDATION_SUBDOMAIN_NAME.DOMAIN_NAME." \
    --ttl="30" \
    --type="CNAME" \
    --zone="DNS_ZONE_NAME"

  gcloud dns record-sets transaction add b31744e5-87ed-4580-8107-92b2231efb5c.0.us-central1.authorize.certificatemanager.goog. \
     --name  _acme-challenge_6wvc2ga4536h76mg.cloudaxt.online. \
    --ttl 30 \
    --type CNAME \
    --zone cloudaxt-online
...

### Execute Transaction Record Sets
...

cloud dns record-sets transaction execute --zone cloudaxt-online

...

### Step-02-03: Create Google SSL certificates with External DNS
...

gcloud certificate-manager certificates create CERTIFICATE_NAME \
    --domains="DOMAIN_NAME, *.DOMAIN_NAME" \
    --dns-authorizations="AUTHORIZATION_NAMES" \
    --location=LOCATION


gcloud certificate-manager certificates create my-regional-certificate \
    --domains *.cloudaxt.online \
    --dns-authorizations dns-authorization-cert \
    --location us-central1 
...

### List Google SSL certificates
...

gcloud certificate-manager certificates list

...

### Describe Google SSL certificates
...

gcloud certificate-manager certificates describe my-regional-certificate  --location us-central1

...

### Add DNS Records

### Start Transaction Record Sets
...

gcloud dns record-sets transaction start --zone cloudaxt-online

...

### Add a A record in Zone
...

gcloud dns record-sets transaction add 35.208.213.11 \
   --name www.cloudaxt.online \
   --ttl 60 \
   --type A \
   --zone cloudaxt-online
...

### Execute Transaction Record Sets
...

cloud dns record-sets transaction execute --zone cloudaxt-online
...

### Add DNS More Records

### Start Transaction Record Sets
...

gcloud dns record-sets transaction start --zone cloudaxt-online

...

### Add a A record in Zone
...

gcloud dns record-sets transaction add 35.208.213.11 \
   --name web1.cloudaxt.online \
   --ttl 60 \
   --type A \
   --zone cloudaxt-online

gcloud dns record-sets transaction add 35.208.213.11 \
   --name web2.cloudaxt.online \
   --ttl 60 \
   --type A \
   --zone cloudaxt-online


...

### Execute Transaction Record Sets
...

cloud dns record-sets transaction execute --zone cloudaxt-online
...


...

                      
```

### Step-02-08: Deploy and Verify Resources
```t
# List Kubernetes Gateway Classes
kubectl get gatewayclass

# Deploy Kubernetes Resources
kubectl apply -f p2-regional-k8sresources-yaml

# List Kubernetes Deployments
kubectl get deploy

# List Kubernetes Pods
kubectl get pods

# List Kubernetes Services
kubectl get svc

# List Kubernetes Gateways created using Gateway API
kubectl get gateway
kubectl get gtw

# Describe Gateway
kubectl describe gateway mygateway1-regional

# List HTTP Route
kubectl get httproute

# Verify Gateway is GCP GKE Console
Go to GKE Console -> Networking -> Gateways, Services & Ingress -> mygateway1-regional

# Verify GCP Cloud Load Balancer
Go to Cloud Load Balancers -> Review load balancer settings

# Host Entry
35.208.69.211 app1.stacksimplify.com
35.208.69.211 app2.stacksimplify.com
35.208.69.211 default.stacksimplify.com

# Access Application
http://<DNS-URL> should redirect to https://<DNS-URL>
App1: http://app1.stacksimplify.com/app1/index.html
App2: http://app2.stacksimplify.com/app2/index.html
App3: http://default.stacksimplify.com
```
### Step-02-07: Clean-up
```t
# Delete Kubernetes Resources
kubectl delete -f p2-regional-k8sresources-yaml
```

## Step-03: Project-3: p3-k8sresources-terraform-manifests: Terraform Manifests
### Step-03-01: NO changes to following manifests
- **Folder:** p3-k8sresources-terraform-manifests
- c2-01-variables.tf
- c2-02-local-values.tf
- c3-01-remote-state-datasource.tf
- c3-02-providers.tf
- c5-01-gateway.tf
- c5-02-gateway-http-to-https-route.tf
- c7-static-ip.tf
- c8-certificate-manager.tf
- terraform.tfvars


### Step-03-02: c1-versions.tf
- Update your Cloud Storage Bucket
```t
  backend "gcs" {
    bucket = "terraform-on-gcp-gke"
    prefix = "dev/k8s-gateway-regional-demo1"    
  }  
```
### Step-03-03: Review MyApp1, MyApp2 and MyApp3 Deployment and Cluster IP Services
1. c4-01-myapp1-deployment.tf
2. c4-02-myapp1-clusterip-service.tf
3. c4-03-myapp2-deployment.tf
4. c4-04-myapp2-clusterip-service.tf
5. c4-05-myapp3-deployment.tf
6. c4-06-myapp3-clusterip-service.tf

### Step-03-04: c5-03-gateway-app1-http-route.tf
```hcl
resource "kubernetes_manifest" "app1_http_route" {
  manifest = {
    apiVersion = "gateway.networking.k8s.io/v1"
    kind       = "HTTPRoute"
    metadata = {
      name = "app1-route"
      namespace = "default"      
    }
    spec = {
      parentRefs = [{
        kind = "Gateway"
        name = "mygateway1-regional"
        sectionName = "https"
      }]
      hostnames = ["app1.stacksimplify.com"]      
      rules = [
        # Rule-1: App1
        {
          matches = [
            {
              path = {
                type  = "PathPrefix"
                value = "/app1"
              }
            }
          ]
          backendRefs = [
            {
              name = kubernetes_service_v1.myapp1_service.metadata[0].name 
              port = 80
            }
          ]
        }
      ]      
    }
  }
}
```
### Step-03-05: c5-04-gateway-app2-http-route.tf
```hcl
resource "kubernetes_manifest" "app2_http_route" {
  manifest = {
    apiVersion = "gateway.networking.k8s.io/v1"
    kind       = "HTTPRoute"
    metadata = {
      name = "app2-route"
      namespace = "default"      
    }
    spec = {
      parentRefs = [{
        kind = "Gateway"
        name = "mygateway1-regional"
        sectionName = "https"
      }]
      hostnames = ["app2.stacksimplify.com"]      
      rules = [
        # Rule-2: App2
        {
          matches = [
            {
              path = {
                type  = "PathPrefix"
                value = "/app2"
              }
            }
          ]
          backendRefs = [
            {
              name = kubernetes_service_v1.myapp2_service.metadata[0].name 
              port = 80
            }
          ]
        }
      ]      
    }
  }
}
```

### Step-03-06: c5-05-gateway-app3-http-route.tf
```hcl
resource "kubernetes_manifest" "app3_http_route" {
  manifest = {
    apiVersion = "gateway.networking.k8s.io/v1"
    kind       = "HTTPRoute"
    metadata = {
      name = "app3-default-route"
      namespace = "default"      
    }
    spec = {
      parentRefs = [{
        kind = "Gateway"
        name = "mygateway1-regional"
        sectionName = "https"
      }]
      rules = [
        # Rule-3: App3 - Default Route
        {
          backendRefs = [
            {
              name = kubernetes_service_v1.myapp3_service.metadata[0].name 
              port = 80
            }
          ]
        }
      ]      
    }
  }
}
```
### Step-03-05: Execute Terraform Commands
```t
# Change Directory
cd p3-regional-k8sresources-terraform-manifests

# Terraform Initialize
terraform init

# Terraform Validate
terraform validate

# Terraform plan
terraform plan

# Terraform Apply
terraform apply -auto-approve
```

### Step-03-06: Verify Kubernetes Resources
```t
# List Kubernetes Deployments
kubectl get deploy

# List Kubernetes Pods
kubectl get pods

# List Kubernetes Services
kubectl get svc

# List Kubernetes Gateways created using Gateway API
kubectl get gateway
kubectl get gtw

# Describe Gateway
kubectl describe gateway mygateway1-regional

# List HTTP Route
kubectl get httproute

# Verify Gateway is GCP GKE Console
Go to GKE Console -> Networking -> Gateways, Services & Ingress -> mygateway1-regional

# Verify GCP Cloud Load Balancer
Go to Cloud Load Balancers -> Review load balancer settings

# Verify Certificate Manager SSL Certificate
Go to Cloud Certificate Manager

# Host Entry
35.208.69.211 web1.chandiolab.site
35.208.69.211 web2.chandiolab.site
35.208.69.211 web3.chandiolab.site


# Access Application
http://<DNS-URL> should redirect to https://<DNS-URL>
App1: http://web1.chandiolab.site
App2: http://web2.chandiolab.site
App3: http://web3.chandiolab.site
```

### Step-03-07: Clean-Up
```t
# Change Directory
cd p3-regional-k8sresources-terraform-manifests

# Terraform Destroy
terraform apply -destroy -auto-approve
```

## Gateway Documentation
- https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.Listener








