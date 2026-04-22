# Google Cloud Load Balancing & Managed Instance Groups Lab

This repository contains the complete sequence of `gcloud` commands required to configure both a Regional Network Load Balancer and a Global HTTP Load Balancer on Google Cloud Platform.

---

## 📂 0. Environment Initialization
Run these commands first to set your default working environment and avoid region/zone errors.

```bash
gcloud config set compute/region us-west4
gcloud config set compute/zone us-west4-c
```

---

## 🖥️ 1. Task 1: Create Web Server Instances
Create three individual Compute Engine VMs and configure the firewall to allow HTTP traffic.

```bash
# Create Instance web1
gcloud compute instances create web1 \
    --zone=us-west4-c \
    --machine-type=e2-small \
    --tags=network-lb-tag \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "<h3>Web Server: web1</h3>" | tee /var/www/html/index.html'

# Create Instance web2
gcloud compute instances create web2 \
    --zone=us-west4-c \
    --machine-type=e2-small \
    --tags=network-lb-tag \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "<h3>Web Server: web2</h3>" | tee /var/www/html/index.html'

# Create Instance web3
gcloud compute instances create web3 \
    --zone=us-west4-c \
    --machine-type=e2-small \
    --tags=network-lb-tag \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "<h3>Web Server: web3</h3>" | tee /var/www/html/index.html'

# Create Firewall Rule to allow HTTP traffic
gcloud compute firewall-rules create www-firewall-network-lb \
    --target-tags=network-lb-tag \
    --allow=tcp:80
```

---

## ⚖️ 2. Task 2: Configure the Network Load Balancing Service
Set up a Layer 4 Regional Load Balancer to distribute traffic among the three web servers.

```bash
# 1. Create a static external IP address
gcloud compute addresses create network-lb-ip-1 --region us-west4

# 2. Create a legacy HTTP health check
gcloud compute http-health-checks create basic-check

# 3. Create the target pool
gcloud compute target-pools create www-pool \
    --region us-west4 \
    --http-health-check basic-check

# 4. Add the instances to the pool
gcloud compute target-pools add-instances www-pool \
    --instances web1,web2,web3 \
    --instances-zone us-west4-c

# 5. Create the forwarding rule
gcloud compute forwarding-rules create www-rule \
    --region us-west4 \
    --ports 80 \
    --address network-lb-ip-1 \
    --target-pool www-pool
```

---

## 🚀 3. Task 3: Create an HTTP Load Balancer
Set up a Layer 7 Global Load Balancer using a Managed Instance Group (MIG).

### Infrastructure Setup
```bash
# 1. Create the Instance Template
gcloud compute instance-templates create lb-backend-template \
   --region=us-west4 \
   --network=default \
   --subnet=default \
   --tags=allow-health-check \
   --machine-type=e2-medium \
   --image-family=debian-12 \
   --image-project=debian-cloud \
   --metadata=startup-script='#!/bin/bash
     apt-get update
     apt-get install apache2 -y
     service apache2 restart
     echo "<h3>Web Server: $(hostname)</h3>" | tee /var/www/html/index.html'

# 2. Create the Managed Instance Group (MIG)
gcloud compute instance-groups managed create lb-backend-group \
   --template=lb-backend-template \
   --size=2 \
   --zone=us-west4-c

# 3. Create Firewall for Google Health Checkers
gcloud compute firewall-rules create fw-allow-health-check \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check \
  --rules=tcp:80
```

### Load Balancer Configuration
```bash
# 4. Reserve Global Static IP
gcloud compute addresses create lb-ipv4-1 --global

# 5. Create Global HTTP Health Check
gcloud compute health-checks create http http-basic-check \
    --global \
    --port 80

# 6. Create Backend Service
gcloud compute backend-services create web-backend-service \
    --protocol=HTTP \
    --health-checks=http-basic-check \
    --global

# 7. Add MIG to the Backend Service
gcloud compute backend-services add-backend web-backend-service \
    --instance-group=lb-backend-group \
    --instance-group-zone=us-west4-c \
    --global

# 8. Create URL Map
gcloud compute url-maps create web-map-http \
    --default-service=web-backend-service

# 9. Create Target HTTP Proxy
gcloud compute target-http-proxies create http-lb-proxy \
    --url-map=web-map-http

# 10. Create Global Forwarding Rule
gcloud compute forwarding-rules create http-content-rule \
    --address=lb-ipv4-1 \
    --global \
    --target-http-proxy=http-lb-proxy \
    --ports=80
```

---

## 🔍 4. Verification & Testing

Use these commands to find your load balancer entry points:

```bash
# Get Network Load Balancer IP
gcloud compute forwarding-rules describe www-rule --region us-west4 --format="value(IPAddress)"

# Get HTTP Load Balancer IP
gcloud compute forwarding-rules describe http-content-rule --global --format="value(IPAddress)"
```

### Note on Propagation
* **Network LB**: Usually active immediately.
* **HTTP LB**: Can take **5-10 minutes** to become active and pass health checks. If you see a `404` or `502`, please wait and refresh.
