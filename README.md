# Secure Workloads in Google Kubernetes Engine: Challenge Lab (GSP335)

![Google Cloud](https://img.shields.io/badge/Google_Cloud-4285F4?logo=googlecloud&logoColor=white)
![Lab](https://img.shields.io/badge/Lab-GSP335-34A853)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white)
![Security](https://img.shields.io/badge/Topic-Container_Security-red)

Reference solution for the Google Cloud Skills Boost challenge lab **Secure Workloads in Google Kubernetes Engine** (GSP335).

## 📖 Overview

You deploy WordPress on GKE backed by Cloud SQL, then harden it in layers:

- **TLS** at the edge with NGINX Ingress and cert-manager (Let's Encrypt).
- **Network policies** to restrict pod traffic.
- **Binary Authorization** so only approved images can run.
- **Pod security policies** to block privileged containers.

## 🧰 Services used

Google Kubernetes Engine · Cloud SQL · Cloud SQL Auth Proxy · Helm · NGINX Ingress · cert-manager · Binary Authorization · IAM

## 📁 Repository structure

```text
.
├── README.md
└── manifests/
    ├── network-policy-allow-nginx.yaml   # Task 4
    └── bin-auth-policy.yaml              # Task 5
```

## ✅ Prerequisites

```bash
export PROJECT_ID=$(gcloud config get-value project)
export ZONE="us-central1-c"
export DB_PASSWORD="<choose-a-strong-password>"

gsutil -m cp gs://cloud-training/gsp335/* .
```

## 🚀 Solution

### Task 1: Create a cluster with network policy enabled

```bash
gcloud container clusters create demo1 \
  --zone $ZONE \
  --machine-type n1-standard-4 \
  --num-nodes 2 \
  --enable-network-policy

gcloud container clusters get-credentials demo1 --zone $ZONE
```

### Task 2: Deploy WordPress with Cloud SQL

```bash
gcloud sql instances create wordpress --region=us-central1
gcloud sql databases create wordpress --instance wordpress
gcloud sql users create wordpress --instance=wordpress --host=% --password="$DB_PASSWORD"

# Service account used by the Cloud SQL Auth Proxy sidecar
gcloud iam service-accounts create sa-wordpress --display-name sa-wordpress
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --role roles/cloudsql.client \
  --member serviceAccount:sa-wordpress@$PROJECT_ID.iam.gserviceaccount.com
gcloud iam service-accounts keys create key.json \
  --iam-account sa-wordpress@$PROJECT_ID.iam.gserviceaccount.com

kubectl create secret generic cloudsql-instance-credentials --from-file key.json
kubectl create secret generic cloudsql-db-credentials \
  --from-literal username=wordpress \
  --from-literal password="$DB_PASSWORD"

kubectl apply -f volume.yaml
sed -i "s/INSTANCE_CONNECTION_NAME/$PROJECT_ID:us-central1:wordpress/g" wordpress.yaml
kubectl apply -f wordpress.yaml
```

### Task 3: Add TLS with NGINX Ingress and cert-manager

```bash
helm repo add stable https://charts.helm.sh/stable
helm repo update
helm install nginx-ingress stable/nginx-ingress

kubectl apply --validate=false \
  -f https://github.com/jetstack/cert-manager/releases/download/v0.16.0/cert-manager.yaml

kubectl get svc   # wait until the ingress controller has an EXTERNAL-IP

sed -i "s/LAB_EMAIL_ADDRESS/sa-wordpress@$PROJECT_ID.iam.gserviceaccount.com/g" issuer.yaml
kubectl apply -f issuer.yaml   # rerun if you see a transient webhook error

. add_ip.sh
HOST_NAME=$(echo $USER | tr -d '_').labdns.xyz
sed -i "s/HOST_NAME/$HOST_NAME/g" ingress.yaml
kubectl apply -f ingress.yaml
```

### Task 4: Restrict traffic with network policies

```bash
kubectl apply -f network-policy.yaml                        # lab-provided deny rules
kubectl apply -f manifests/network-policy-allow-nginx.yaml  # allow internet → ingress controller
```

### Task 5: Enforce Binary Authorization

```bash
gcloud services enable \
  container.googleapis.com \
  containeranalysis.googleapis.com \
  binaryauthorization.googleapis.com

gcloud container clusters update demo1 --enable-binauthz --zone $ZONE

gcloud container binauthz policy export > bin-auth-policy.yaml
# Merge the allow-list and default rule from manifests/bin-auth-policy.yaml, then:
gcloud container binauthz policy import bin-auth-policy.yaml
```

### Task 6: Apply pod security policies

```bash
kubectl apply -f psp-restrictive.yaml
kubectl apply -f psp-role.yaml
kubectl apply -f psp-use.yaml
```

## 🔎 Version notes

- The lab uses the archived Helm `stable/nginx-ingress` chart, because its pod labels match the lab's network policy and Binary Authorization allow-list. Outside the lab, use the official `ingress-nginx` chart and adjust those labels and image patterns.
- PodSecurityPolicy was removed in Kubernetes 1.25. On current clusters, use **Pod Security Admission** labels instead.
- Avoid hard-coding database passwords. Store them in a variable or Secret Manager as shown.

---

## ⚠️ Notes

- Resource names, zones, regions and ports are generated per lab session. Always use the values shown in **your** lab instructions.
- These steps were written when the lab was live. Google revises labs over time, so the current version may differ.

## 📌 Disclaimer

This repository is a study reference. Attempt the lab on your own first, then compare approaches. It is not affiliated with or endorsed by Google.

## 👤 Author

**Girish Sharma** · Cloud & DevOps Engineer · [GitHub](https://github.com/codewithgarry) · [LinkedIn](https://www.linkedin.com/in/codewithgarry/)
