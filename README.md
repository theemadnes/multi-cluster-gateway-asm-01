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

# set up ingress gateways across both clusters
kubectl --context ${CLUSTER_1} create namespace ${IG_NAMESPACE}
kubectl --context ${CLUSTER_1} label namespace ${IG_NAMESPACE} istio-injection=enabled
kubectl --context ${CLUSTER_2} create namespace ${IG_NAMESPACE}
kubectl --context ${CLUSTER_2} label namespace ${IG_NAMESPACE} istio-injection=enabled

mkdir -p asm-ig/base

cat <<EOF > asm-ig/base/kustomization.yaml
resources:
  - github.com/GoogleCloudPlatform/anthos-service-mesh-samples/docs/ingress-gateway-asm-manifests/base
EOF

mkdir asm-ig/variant

cat <<EOF > asm-ig/variant/role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: asm-ingressgateway
  namespace: ${IG_NAMESPACE}
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
EOF

cat <<EOF > asm-ig/variant/rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: asm-ingressgateway
  namespace: ${IG_NAMESPACE}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: asm-ingressgateway
subjects:
  - kind: ServiceAccount
    name: asm-ingressgateway
EOF

cat <<EOF > asm-ig/variant/service-proto-type.yaml 
apiVersion: v1
kind: Service
metadata:
  name: asm-ingressgateway
spec:
  ports:
  - name: status-port
    port: 15021
    protocol: TCP
    targetPort: 15021
  - name: http
    port: 80
    targetPort: 8080
    appProtocol: HTTP
  - name: https
    port: 443
    targetPort: 8443
    appProtocol: HTTP2
  type: ClusterIP
EOF

cat <<EOF > asm-ig/variant/gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: asm-ingressgateway
spec:
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*" # IMPORTANT: Must use wildcard here when using SSL, see note below
    #tls:
    #  mode: SIMPLE
    #  credentialName: edge2mesh-credential
EOF

cat <<EOF > asm-ig/variant/kustomization.yaml 
namespace: ${IG_NAMESPACE}
resources:
- ../base
- role.yaml
- rolebinding.yaml
patches:
- path: service-proto-type.yaml
  target:
    kind: Service
- path: gateway.yaml
  target:
    kind: Gateway
EOF

# apply IG specs
kubectl --context ${CLUSTER_1} apply -k asm-ig/variant
kubectl --context ${CLUSTER_2} apply -k asm-ig/variant
```