```
# ENV Setup

export GCP_PROJECT_ID=data-protection-01
gcloud config set project $GCP_PROJECT_ID
gcloud config set compute/region us-east1
gcloud config set compute/zone us-east1-b
export GKE_CLUSTER_NAME=dmilan-gke-01
export GKE_NAMESPACE=identest
export GSA_ID=workload-identity-test-gsa
export KSA_ID=workload-identity-test-ksa


# Create GSA Account

gcloud iam service-accounts create ${GSA_ID}
gcloud projects add-iam-policy-binding $GCP_PROJECT_ID \
  --member serviceAccount:${GSA_ID}@${GCP_PROJECT_ID}.iam.gserviceaccount.com \
  --role roles/storage.objectViewer

# Create GKE Cluster


# gcloud beta container clusters create $GKE_CLUSTER_NAME \
#  --cluster-version=1.14 \
#  --identity-namespace=$GCP_PROJECT_ID.svc.id.goog

gcloud container clusters get-credentials $GKE_CLUSTER_NAME
kubectl create namespace ${GKE_NAMESPACE}
kubectl create serviceaccount --namespace ${GKE_NAMESPACE} ${KSA_ID}

gcloud iam service-accounts add-iam-policy-binding \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:${GCP_PROJECT_ID}.svc.id.goog[${GKE_NAMESPACE}/${KSA_ID}]" \
  ${GSA_ID}@${GCP_PROJECT_ID}.iam.gserviceaccount.com


kubectl annotate serviceaccount \
  --namespace ${GKE_NAMESPACE} \
  ${KSA_ID} \
  iam.gke.io/gcp-service-account=workload-identity-test@${GCP_PROJECT_ID}.iam.gserviceaccount.com

kubectl run --rm -it \
  --generator=run-pod/v1 \
  --image google/cloud-sdk:slim \
  --serviceaccount ${KSA_ID} \
  --namespace ${GKE_NAMESPACE} \
  test-pod

# kubectl run -it --rm --restart='Never' --image=gcr.io/data-protection-01/alpine-plus:latest --serviceaccount ${KSA_ID} --namespace ${GKE_NAMESPACE} alpine-plus

gcloud auth list

# Clean Up

gcloud beta container clusters delete $GKE_CLUSTER_NAME -q
gcloud iam service-accounts delete ${GSA_ID}@${GCP_PROJECT_ID}.iam.gserviceaccount.com -q
```
