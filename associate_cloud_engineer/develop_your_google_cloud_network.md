# Develop your Google Cloud Network: Challenge Lab (GSP321)

This guide provides the complete, production-ready implementation workflow to successfully pass all 9 validation checkpoints for the GSP321 Challenge Lab.

---

## 🛠️ Phase 1: Environment Initialization

> ⚠️ **Crucial Operational Rule**: Always log into the Google Cloud Console using **Username 1**. **Username 2** is exclusively reserved for the access delegation step in Task 9.

Open **Cloud Shell** and execute the following commands to initialize the environment architecture:

```bash
export REGION=us-central1
export ZONE=us-central1-b
export PROJECT_ID=\((gcloud config get-value project 2>/dev/null \vert{}\vert{} echo\)GOOGLE_CLOUD_PROJECT)
```

---

## 🌐 Phase 2: Custom Network Topography Setup

### Task 1. Create Development VPC Manually
```bash
gcloud compute networks create griffin-dev-vpc --subnet-mode=custom

gcloud compute networks subnets create griffin-dev-wp \
    --network=griffin-dev-vpc \
    --region=\$REGION \
    --range=192.168.16.0/20

gcloud compute networks subnets create griffin-dev-mgmt \
    --network=griffin-dev-vpc \
    --region=\$REGION \
    --range=192.168.32.0/20
```

### Task 2. Create Production VPC Manually
```bash
gcloud compute networks create griffin-prod-vpc --subnet-mode=custom

gcloud compute networks subnets create griffin-prod-wp \
    --network=griffin-prod-vpc \
    --region=\$REGION \
    --range=192.168.48.0/20

gcloud compute networks subnets create griffin-prod-mgmt \
    --network=griffin-prod-vpc \
    --region=\$REGION \
    --range=192.168.64.0/20
```

### Task 3. Create Bastion Host
Provision a dual-homed secure administration instance and configure custom firewall access rule patterns:
```bash
gcloud compute instances create bastion \
    --zone=\$ZONE \
    --machine-type=e2-medium \
    --network-interface=subnet=griffin-dev-mgmt \
    --network-interface=subnet=griffin-prod-mgmt

gcloud compute firewall-rules create dev-bastion-ssh \
    --network=griffin-dev-vpc \
    --allow=tcp:22 \
    --source-ranges=0.0.0.0/0

gcloud compute firewall-rules create prod-bastion-ssh \
    --network=griffin-prod-vpc \
    --allow=tcp:22 \
    --source-ranges=0.0.0.0/0
```

---

## 🗄️ Phase 3: Database & Container Architecture

### Task 4. Create and Configure Cloud SQL Instance
1. Provision the managed MySQL server layout (this process takes roughly **4-5 minutes**):
```bash
gcloud sql instances create griffin-dev-db \
    --database-version=MYSQL_8_0 \
    --cpu=2 \
    --memory=7680MB \
    --region=\$REGION \
    --root-password=stormwind_rules
```
2. Connect to the active instance terminal layer:
```bash
gcloud sql connect griffin-dev-db --user=root
```
3. Type `stormwind_rules` when prompted for the password. Paste the following SQL structural setup block and hit Enter:
```sql
CREATE DATABASE wordpress;
CREATE USER "wp_user"@"%" IDENTIFIED BY "stormwind_rules";
GRANT ALL PRIVILEGES ON wordpress.* TO "wp_user"@"%";
FLUSH PRIVILEGES;
EXIT;
```

### Task 5. Create Kubernetes Cluster
```bash
gcloud container clusters create griffin-dev \
    --network=griffin-dev-vpc \
    --subnetwork=griffin-dev-wp \
    --zone=\$ZONE \
    --num-nodes=2 \
    --machine-type=e2-standard-4
```

### Task 6. Prepare the Kubernetes Cluster
Download variables, strip out configuration placeholders, and securely structure access credentials to prevent "Already Exists" conflicts:
```bash
gsutil cp -r gs://spls/gsp321/wp-k8s .
cd wp-k8s

# Inject database credentials
sed -i "s/username_goes_here/wp_user/g" wp-env.yaml
sed -i "s/password_goes_here/stormwind_rules/g" wp-env.yaml

# Apply configuration
kubectl apply -f wp-env.yaml

# Idempotent Key generation & creation pipeline
rm -f key.json
kubectl delete secret cloudsql-instance-credentials --ignore-not-found=true

gcloud iam service-accounts keys create key.json \
    --iam-account=cloud-sql-proxy@\${PROJECT_ID}.iam.gserviceaccount.com

kubectl create secret generic cloudsql-instance-credentials \
    --from-file=key.json
```

---

## 🚀 Phase 4: Application Deployment & Monitoring

### Task 7. Create a WordPress Deployment
This automated block extracts your unique database instance proxy string, builds the container layout, and spins up your public endpoint balancer:
```bash
# Extract dynamic runtime parameters
export DB_CONNECTION=\$(gcloud sql instances describe griffin-dev-db --format="value(connectionName)")

# Generate explicit deployment layout
cat <<EOF > wp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
        - image: wordpress:5.7.2-apache
          name: wordpress
          env:
          - name: WORDPRESS_DB_HOST
            value: 127.0.0.1:3306
          - name: WORDPRESS_DB_USER
            valueFrom:
              secretKeyRef:
                name: mysql-credentials
                key: username
          - name: WORDPRESS_DB_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mysql-credentials
                key: password
          ports:
            - containerPort: 80
              name: wordpress
          volumeMounts:
            - name: wordpress-persistent-storage
              mountPath: /var/view/html
        - name: cloud-sql-proxy
          image: gcr.io/cloudsql-docker/gce-proxy:1.22.0
          command:
            - "/cloud_sql_proxy"
            - "-instances=\${DB_CONNECTION}=tcp:3306"
            - "-credential_file=/secrets/cloudsql/key.json"
          securityContext:
            runAsNonRoot: true
          volumeMounts:
            - name: cloudsql-instance-credentials
              mountPath: /secrets/cloudsql
              readOnly: true
      volumes:
        - name: wordpress-persistent-storage
          persistentVolumeClaim:
            claimName: wp-pv-claim
        - name: cloudsql-instance-credentials
          secret:
            secretName: cloudsql-instance-credentials
EOF

# Launch containers
kubectl apply -f wp-deployment.yaml
kubectl apply -f wp-service.yaml

# Fetch External IP (Copy this IP address as soon as it populates!)
kubectl get svc --watch
```
> **Note**: Press `CTRL + C` once the public `EXTERNAL-IP` address appears to free up your terminal line.

### Task 8. Enable Monitoring
1. Search for **Monitoring** in the Google Cloud Console search bar.
2. Select **Uptime checks** from the left navigation and click **+ Create Uptime Check**.
3. Use the following metrics:
   * **Title**: `WordPress Uptime`
   * **Protocol**: `HTTP`
   * **Resource Type**: `URL`
   * **Hostname**: `[Paste your WordPress External IP here]`
   * **Path**: `/`
4. Click **Create**.

### Task 9. Provide Access for an Additional Engineer
1. Copy the **Username 2** email address provided in your lab credentials sidebar panel.
2. Navigate to **IAM & Admin** > **IAM** in the Cloud Console dashboard.
3. Click **+ Grant Access** at the top of the IAM page.
4. Paste the **Username 2** email into the **New principals** text box.
5. Set the role dropdown to **Project** > **Editor**.
6. Click **Save**.
