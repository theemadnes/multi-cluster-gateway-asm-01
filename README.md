# multi-cluster-gateway-asm-01
example of setting up 2 GKE clusters w/ multi-cluster gateways and Anthos Service Mesh

some notes to review before proceeding:
- this demo uses GKE autopilot
- as of this writing (late June 2023), multi-cluster gateway classes don't yet support TLS to the backend, so the connection between the Google load balancer and the ASM ingress gateways will be HTTP, not HTTP(S) or HTTP/2 using TLS. Fear not, because the connection is already encrypted at the network layer, *but* for those that want to control the keys used for encryption, TLS support is coming soon
- this demo uses a VPC named `default`
- review the `envvars` section below and create 2x GKE autopilot clusters using the names provided (`autopilot-cluster-us-central1` and `autopilot-cluster-us-east4`) in the correct regions, and make sure to enable Anthos Service Mesh on both clusters when creating them 
- make sure your clusters are using GKE version 1.26 or later to enable the Gateway API. If you've created them and they aren't of the required minimum version, perform a cluster upgrade to bring them both up to what we need for this walkthrough

### instructions

```
# envvars
export PROJECT=gateway-multicluster-01 # replace with your own project
export PROJECT_NUMBER=$(gcloud projects list \
--filter="${PROJECT}" \
--format="value(PROJECT_NUMBER)")
export IG_NAMESPACE=asm-ingress
export REGION_1=us-central1
export REGION_2=us-east4
export CLUSTER_1=autopilot-cluster-us-central1 # designated as config cluster
export CLUSTER_2=autopilot-cluster-us-east4
export CLUSTER_VERSION=1.26 # replace with your version
export RELEASE_CHANNEL=REGULAR # replace with your release channel
export PUBLIC_ENDPOINT=frontend.endpoints.${PROJECT}.cloud.goog

# enable APIs 
gcloud services enable \
  container.googleapis.com \
  gkehub.googleapis.com \
  multiclusterservicediscovery.googleapis.com \
  multiclusteringress.googleapis.com \
  trafficdirector.googleapis.com \
  --project=${PROJECT}

# enable multi-cluster services (MCS)
gcloud container fleet multi-cluster-services enable \
    --project ${PROJECT}

# grant MCS IAM permissions to the project
gcloud projects add-iam-policy-binding ${PROJECT} \
    --member "serviceAccount:${PROJECT}.svc.id.goog[gke-mcs/gke-mcs-importer]" \
    --role "roles/compute.networkViewer" \
    --project=${PROJECT}

# add clusters to kubeconfig
gcloud container clusters get-credentials ${CLUSTER_1} --region ${REGION_1} --project ${PROJECT}
gcloud container clusters get-credentials ${CLUSTER_2} --region ${REGION_2} --project ${PROJECT}

# rename kubeconfig entries to make them a bit easier to reference
kubectl config rename-context gke_${PROJECT}_${REGION_1}_${CLUSTER_1} ${CLUSTER_1}
kubectl config rename-context gke_${PROJECT}_${REGION_2}_${CLUSTER_2} ${CLUSTER_2}

# designate CLUSTER_1 as the config cluster
gcloud container fleet ingress enable \
    --config-membership=${CLUSTER_1} \
    --project=${PROJECT}

# verify Gateway Controller is enabled for the fleet
gcloud container fleet ingress describe --project=${PROJECT}

# grant IAM permissions for the Gateway Controller
gcloud projects add-iam-policy-binding ${PROJECT} \
    --member "serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-multiclusteringress.iam.gserviceaccount.com" \
    --role "roles/container.admin" \
    --project=${PROJECT}

# verify that your clusters have the `gatewayclass`es enabled
kubectl get gatewayclass --context=$CLUSTER_1
kubectl get gatewayclass --context=$CLUSTER_2

# just in case the `gatewayclass`es aren't enabled, force install them on both clusters
gcloud container clusters update ${CLUSTER_1} --region=${REGION_1}\
  --gateway-api=standard \
  --project ${PROJECT}
gcloud container clusters update ${CLUSTER_2} --region=${REGION_2}\
  --gateway-api=standard \
  --project ${PROJECT}
```