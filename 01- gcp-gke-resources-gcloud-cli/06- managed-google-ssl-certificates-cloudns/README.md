---
title: GKE with Kubernetes Gateway API - Google Managed Certificates
description: Create GCP Application Load Balancer using GKE Kubernetes Gateway API with Domain Name based routing
---

## Step-01: Introduction
1. Create GCP Application Load Balancer using Kubernetes Gateway API, Static IP for Load Balancer, Self-signed SSL with Google Cloud Certificate Manager, Domain Name based routing 
2. **Approach-1:** Using Kubernetes YAML Manifests
3. **Approach-2:** Using Terraform Manifests

# Cloud DNS -  Basics

## Step-01-01: Introduction
1. Register a Domain using Cloud Domains
2. Create a Cloud DNS Zone
3. Reserve the External IP Address
4. Create VM Instance with sample app, reserved external IP
5. Create DNS Record Set
6. Access Sample Application using browser with DNS Name
7. Delete all the resources created as part of this demo

## Step-01-02: Review/Create Cloud DNS Zone 
- Goto Network Services -> Cloud DNS -> CREATE ZONE
- **Zone type:** Public
- **Zone name:** cloudxt-online
- **DNS Name:** cloudxt.online
- REST ALL LEAVE TO DEFAULTS
- Click on **CREATE**

### Step-02-01: Create Regional Static IP
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

### Step-02-01: Create Global Static IP
```t
# Create Global Load Balancer IP
gcloud compute addresses create my-global-pip --global


# List IP Addresss    
gcloud compute addresses list
gcloud compute addresses describe my-global-pip --global
```

### Step-02-02: Create Google  SSL certificates
```t
# Create DNS Authorization 
gcloud certificate-manager dns-authorizations create dns-authorization-cert \
    --domain cloudaxt.online \
    --type=PER_PROJECT_RECORD \
    --location us-central1 

### List DNS Authorization 
 
gcloud certificate-manager dns-authorizations list

gcloud certificate-manager dns-authorizations describe dns-authorization-cert --location us-central1

data: b31744e5-87ed-4580-8107-92b2231efb5c.0.us-central1.authorize.certificatemanager.goog.
name:  _acme-challenge_6wvc2ga4536h76mg.cloudaxt.online.


### Start Transaction Record Sets

gcloud dns record-sets transaction start --zone cloudaxt-online



### Add Transaction Record Sets

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


### Execute Transaction Record Sets

cloud dns record-sets transaction execute --zone cloudaxt-online

### Step-02-03: Create Google SSL certificates with External DNS

gcloud certificate-manager certificates create CERTIFICATE_NAME \
    --domains="DOMAIN_NAME, *.DOMAIN_NAME" \
    --dns-authorizations="AUTHORIZATION_NAMES" \
    --location=LOCATION


gcloud certificate-manager certificates create my-regional-certificate \
    --domains *.cloudaxt.online \
    --dns-authorizations dns-authorization-cert \
    --location us-central1 

### List Google SSL certificates


gcloud certificate-manager certificates list


### Describe Google SSL certificates

gcloud certificate-manager certificates describe my-regional-certificate  --location us-central1

### Add DNS Records

### Start Transaction Record Sets


gcloud dns record-sets transaction start --zone cloudaxt-online


### Add a A record in Zone

gcloud dns record-sets transaction add 35.208.213.11 \
   --name www.cloudaxt.online \
   --ttl 60 \
   --type A \
   --zone cloudaxt-online

### Execute Transaction Record Sets


cloud dns record-sets transaction execute --zone cloudaxt-online

### Add DNS More Records

### Start Transaction Record Sets

gcloud dns record-sets transaction start --zone cloudaxt-online

### Add a A record in Zone


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


### Execute Transaction Record Sets

cloud dns record-sets transaction execute --zone cloudaxt-online
                      
```

