```
# ENV Setup

export GCP_PROJECT_ID=jenkins-on-kubernetes-01
gcloud config set project $GCP_PROJECT_ID
gcloud config set compute/region us-east1
gcloud config set compute/zone us-east1-b

# Create GKE Cluster

gcloud iam service-accounts create workload-identity-test
gcloud projects add-iam-policy-binding $GCP_PROJECT_ID \
  --member serviceAccount:workload-identity-test@${GCP_PROJECT_ID}.iam.gserviceaccount.com \
  --role roles/storage.objectViewer

export GKE_CLUSTER_NAME=gke-wi

gcloud beta container clusters create $GKE_CLUSTER_NAME \
  --cluster-version=1.14 \
  --identity-namespace=$GCP_PROJECT_ID.svc.id.goog

gcloud container clusters get-credentials $GKE_CLUSTER_NAME
kubectl create namespace newspace
kubectl create serviceaccount --namespace newspace workload-identity-test-ksa

gcloud iam service-accounts add-iam-policy-binding \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:${GCP_PROJECT_ID}.svc.id.goog[newspace/workload-identity-test-ksa]" \
  workload-identity-test@${GCP_PROJECT_ID}.iam.gserviceaccount.com


kubectl annotate serviceaccount \
  --namespace newspace \
  workload-identity-test-ksa \
  iam.gke.io/gcp-service-account=workload-identity-test@${GCP_PROJECT_ID}.iam.gserviceaccount.com

kubectl run --rm -it \
  --generator=run-pod/v1 \
  --image google/cloud-sdk:slim \
  --serviceaccount workload-identity-test-ksa \
  --namespace newspace \
  test-pod


# Clean Up

gcloud beta container clusters delete $GKE_CLUSTER_NAME -q
gcloud iam service-accounts delete workload-identity-test@$GCP_PROJECT_ID.iam.gserviceaccount.com -q
```
