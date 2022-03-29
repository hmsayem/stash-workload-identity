# Stash Backup Using Workload Identity  

## Google Kubernetes Engine (GKE)

### Before You Begin

- Ensure that you have enabled the [Google Kubernetes Engine API](https://console.cloud.google.com/flows/enableapi?apiid=container.googleapis.com).
- Ensure that you have installed the [Google Cloud CLI](https://cloud.google.com/sdk/downloads).
- Set up default Google Cloud CLI settings for your project by using one of the following methods:
  - Use `gcloud init`, if you want to be walked through setting project defaults.
  - Use `gcloud config`, to individually set your project ID, zone, and region.
- Ensure that you have enabled the [IAM Service Account Credentials API](https://console.cloud.google.com/apis/api/iamcredentials.googleapis.com/overview).
- Ensure that you have the following IAM roles:
  - `roles/container.admin`
  - `roles/iam.serviceAccountAdmin`

### Create Cluster

```bash
gcloud container clusters create CLUSTER_NAME \
    --region=COMPUTE_REGION \
    --workload-pool=PROJECT_ID.svc.id.goog
```

### Backup Using Workload Identity

#### Get Credentials for the Cluster:

```bash
gcloud container clusters get-credentials CLUSTER_NAME
```

#### Install Stash

```bash
$ helm repo add appscode https://charts.appscode.com/stable/
$ helm repo update
$ helm install stash appscode/stash \
  --version v2022.02.22 \
  --namespace stash --create-namespace \
  --set features.enterprise=true \
  --set-file global.license=/path/to/the/license.txt
```

#### Install Kubedb

```bash
$ helm repo add appscode https://charts.appscode.com/stable/
$ helm repo update
$ helm install kubedb appscode/kubedb \
  --version v2022.03.28 \
  --namespace kubedb --create-namespace \
  --set kubedb-provisioner.enabled=true \
  --set kubedb-ops-manager.enabled=true \
  --set kubedb-autoscaler.enabled=true \
  --set kubedb-dashboard.enabled=true \
  --set kubedb-schema-manager.enabled=true \
  --set-file global.license=/path/to/the/license.txt
```

#### Create an IAM service account

```bash
gcloud iam service-accounts create GSA_NAME \
    --project=GSA_PROJECT
```

#### Ensure the Roles of the IAM Service Account

```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member "serviceAccount:GSA_NAME@GSA_PROJECT.iam.gserviceaccount.com" \
    --role "roles/storage.objectAdmin"
```

```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member "serviceAccount:GSA_NAME@GSA_PROJECT.iam.gserviceaccount.com" \
    --role "roles/storage.admin"
```

#### IAM Policy Binding

```bash
gcloud iam service-accounts add-iam-policy-binding GSA_NAME@GSA_PROJECT.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:PROJECT_ID.svc.id.goog[NAMESPACE/BACKUP_CONFIGURATION_SERVICE_ACCOUNT]"
```

```bash
gcloud iam service-accounts add-iam-policy-binding GSA_NAME@GSA_PROJECT.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:PROJECT_ID.svc.id.goog[NAMESPACE/RESTORE_SESSION_SERVICE_ACCOUNT]"
```

This binding allows the Kubernetes service account to act as the IAM service account.

#### Prepare Backup

- Deploy a Database
- Create a Secret with Restic Password
- Create a Repository

#### Provide NodeSelector and Annotations

Provide the the nodeSelector and anntations in the `spec.runtimeSettings.pod` field of the `BackupConfiguration` and `RestoreSession`.

```yaml
spec:
  runtimeSettings:
    pod:
      nodeSelector:
        iam.gke.io/gke-metadata-server-enabled: "true"
      serviceAccountAnnotations: 
        iam.gke.io/gcp-service-account: "GSA_NAME@GSA_PROJECT.iam.gserviceaccount.com"
```
