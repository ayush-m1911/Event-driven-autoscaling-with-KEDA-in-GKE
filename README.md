# 🚀 Event-Driven Autoscaling with KEDA on GKE

This repository demonstrates a practical implementation of **event-driven autoscaling in Kubernetes** using KEDA on Google Kubernetes Engine (GKE).

It includes a sample application along with deployment templates to showcase how workloads can scale dynamically based on external events such as Pub/Sub messages.

---

## 📌 Overview

This project shows how to:

* Configure **event-driven autoscaling** using KEDA
* Integrate **Google Cloud Pub/Sub** as an event source
* Use **Workload Identity** for secure access to GCP services
* Automatically scale Kubernetes workloads based on message load

---

## 📂 Project Structure

```
app/         → Sample application source code  
templates/   → Kubernetes manifests for deployment and autoscaling  
```

---

## ⚙️ Prerequisites

* Google Kubernetes Engine (GKE) cluster with Workload Identity enabled
* `kubectl` installed
* `helm` installed
* `gcloud` CLI configured

---

## 🔧 Setup Pub/Sub Resources

```bash
GCP_PROJECT_ID=$(gcloud config get-value project)
TOPIC_NAME=keda-demo-topic
SUBSCRIPTION_NAME=keda-demo-topic-subscription

# Create Topic
gcloud pubsub topics create $TOPIC_NAME --project $GCP_PROJECT_ID

# Create Subscription
gcloud pubsub subscriptions create $SUBSCRIPTION_NAME \
--topic $TOPIC_NAME \
--project $GCP_PROJECT_ID
```

---

## 🔐 Configure Workload Identity for KEDA

Workload Identity allows Kubernetes workloads to securely access GCP services without using service account keys.

```bash
KEDA_GCP_SERVICE_ACCOUNT=keda-operator
KEDA_NAMESPACE=keda
KEDA_K8S_SERVICE_ACCOUNT=keda-operator

# Create GCP service account
gcloud iam service-accounts create $KEDA_GCP_SERVICE_ACCOUNT \
--project=$GCP_PROJECT_ID

# Assign role
gcloud projects add-iam-policy-binding $GCP_PROJECT_ID \
--member "serviceAccount:$KEDA_GCP_SERVICE_ACCOUNT@$GCP_PROJECT_ID.iam.gserviceaccount.com" \
--role "roles/monitoring.viewer"

# Allow Kubernetes service account to impersonate GCP account
gcloud iam service-accounts add-iam-policy-binding \
$KEDA_GCP_SERVICE_ACCOUNT@$GCP_PROJECT_ID.iam.gserviceaccount.com \
--role roles/iam.workloadIdentityUser \
--member "serviceAccount:$GCP_PROJECT_ID.svc.id.goog[$KEDA_NAMESPACE/$KEDA_K8S_SERVICE_ACCOUNT]"
```

---

## 📦 Install KEDA

Add Helm repository and install KEDA:

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

helm upgrade --install keda kedacore/keda \
--namespace keda \
--create-namespace \
--set serviceAccount.annotations."iam\.gke\.io/gcp-service-account"="$KEDA_GCP_SERVICE_ACCOUNT@$GCP_PROJECT_ID.iam.gserviceaccount.com" \
--wait
```

> Ensure firewall rules allow traffic on port **9443** for KEDA webhook communication.

---

## 🚀 Deploy Sample Application

```bash
SAMPLE_APP_GCP_SERVICE_ACCOUNT=keda-demo
SAMPLE_APP_NAMESPACE=default
SAMPLE_APP_K8S_SERVICE_ACCOUNT=keda-demo

# Create service account
gcloud iam service-accounts create $SAMPLE_APP_GCP_SERVICE_ACCOUNT \
--project=$GCP_PROJECT_ID

# Assign Pub/Sub role
gcloud projects add-iam-policy-binding $GCP_PROJECT_ID \
--member "serviceAccount:$SAMPLE_APP_GCP_SERVICE_ACCOUNT@$GCP_PROJECT_ID.iam.gserviceaccount.com" \
--role "roles/pubsub.subscriber"
```

Deploy application:

```bash
kubectl apply -f deployment.yaml
```

---

## 📈 Configure KEDA Autoscaling

KEDA uses **ScaledObject** and **TriggerAuthentication** to define scaling behavior.

```bash
kubectl apply -f keda-scaler.yaml
```

### Key Configuration

* `pollingInterval`: 5 seconds
* `minReplicaCount`: 0
* `maxReplicaCount`: 10
* Scaling trigger: Pub/Sub subscription size

---

## 🧪 Testing

Generate messages to trigger scaling:

```bash
./generate-message.sh
```

### Expected Behavior

* Pods scale **up** as message count increases
* Pods scale **down to 0** when queue is empty

---

## 🎯 Key Concepts Covered

* Event-driven autoscaling
* Kubernetes custom resources (CRDs)
* Pub/Sub-based scaling
* Secure identity management using Workload Identity
* Horizontal Pod Autoscaler (HPA) integration

---

## 📜 Reference

Based on concepts from:
https://www.doit.com/event-driven-autoscaling-in-kubernetes-harnessing-the-power-of-keda/

---

## 🤝 Contribution

Feel free to fork the repository and contribute improvements.

---

## 📄 License

This project is intended for educational and demonstration purposes.
