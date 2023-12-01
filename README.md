# multi-cluster-gateway-asm-01
example of setting up 2 GKE clusters w/ multi-cluster gateways and Anthos Service Mesh

some notes to review before proceeding:
- this demo uses GKE autopilot
- this demo uses a VPC named `default`
- because we're doing multi-cluster, make sure the FW rules within your VPC permit things to work correctly. check [this](https://cloud.google.com/service-mesh/docs/unified-install/gke-install-multi-cluster#create_firewall_rule) doc.

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
  certificatemanager.googleapis.com \
  --project=${PROJECT}

# enable multi-cluster services (MCS)
gcloud container fleet multi-cluster-services enable \
    --project ${PROJECT}

# grant MCS IAM permissions to the project
gcloud projects add-iam-policy-binding ${PROJECT} \
    --member "serviceAccount:${PROJECT}.svc.id.goog[gke-mcs/gke-mcs-importer]" \
    --role "roles/compute.networkViewer" \
    --project=${PROJECT}

# create clusters
gcloud container --project ${PROJECT} clusters create-auto ${CLUSTER_1} --region ${REGION_1} --release-channel rapid
gcloud container --project ${PROJECT} clusters create-auto ${CLUSTER_2} --region ${REGION_2} --release-channel rapid

# rename kubeconfig entries to make them a bit easier to reference
kubectl config rename-context gke_${PROJECT}_${REGION_1}_${CLUSTER_1} ${CLUSTER_1}
kubectl config rename-context gke_${PROJECT}_${REGION_2}_${CLUSTER_2} ${CLUSTER_2}

# enable mesh on the fleet
gcloud --project ${PROJECT} container fleet mesh enable

# register clusters to the fleet
gcloud --project ${PROJECT} container fleet memberships register ${CLUSTER_1} --gke-cluster ${REGION_1}/${CLUSTER_1}
gcloud --project ${PROJECT} container fleet memberships register ${CLUSTER_2} --gke-cluster ${REGION_2}/${CLUSTER_2}

# apply `mesh_id` label to the clusters
gcloud container clusters update ${CLUSTER_1} --project ${PROJECT} --region ${REGION_1} --update-labels mesh_id=proj-${PROJECT_NUMBER}
gcloud container clusters update ${CLUSTER_2} --project ${PROJECT} --region ${REGION_2} --update-labels mesh_id=proj-${PROJECT_NUMBER}

# enable service mesh managed control plane and managed data plane
gcloud --project ${PROJECT} container fleet mesh update --management automatic --memberships ${CLUSTER_1}
gcloud --project ${PROJECT} container fleet mesh update --management automatic --memberships ${CLUSTER_2}

# wait a few minutes and verify that the mesh control plane statuses are `ACTIVE`
gcloud --project ${PROJECT} container fleet mesh describe

# designate CLUSTER_1 as the config cluster for load balancing
gcloud container fleet ingress enable \
    --config-membership=${CLUSTER_1} \
    --location=${REGION_1} \
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

# set up ingress gateways across both clusters
kubectl --context ${CLUSTER_1} create namespace ${IG_NAMESPACE}
kubectl --context ${CLUSTER_1} label namespace ${IG_NAMESPACE} istio-injection=enabled
kubectl --context ${CLUSTER_2} create namespace ${IG_NAMESPACE}
kubectl --context ${CLUSTER_2} label namespace ${IG_NAMESPACE} istio-injection=enabled

# create self-signed cert for TLS termination between GFE and ingress gateways
openssl req -new -newkey rsa:4096 -days 365 -nodes -x509 \
 -subj "/CN=frontend.endpoints.${PROJECT}.cloud.goog/O=Edge2Mesh Inc" \
 -keyout frontend.endpoints.${PROJECT}.cloud.goog.key \
 -out frontend.endpoints.${PROJECT}.cloud.goog.crt

kubectl --context ${CLUSTER_1} -n asm-ingress create secret tls edge2mesh-credential \
 --key=frontend.endpoints.${PROJECT}.cloud.goog.key \
 --cert=frontend.endpoints.${PROJECT}.cloud.goog.crt

kubectl --context ${CLUSTER_2} -n asm-ingress create secret tls edge2mesh-credential \
 --key=frontend.endpoints.${PROJECT}.cloud.goog.key \
 --cert=frontend.endpoints.${PROJECT}.cloud.goog.crt

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
  namespace: ${IG_NAMESPACE}
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
  namespace: ${IG_NAMESPACE}
spec:
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "*" # IMPORTANT: Must use wildcard here when using SSL, see note below
    tls:
      mode: SIMPLE
      credentialName: edge2mesh-credential
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

# create service exports for the ingress gateways in each cluster
cat <<EOF > svc_export.yaml 
kind: ServiceExport
apiVersion: net.gke.io/v1
metadata:
  name: asm-ingressgateway
  namespace: ${IG_NAMESPACE}
EOF

kubectl --context=${CLUSTER_1} apply -f svc_export.yaml
kubectl --context=${CLUSTER_2} apply -f svc_export.yaml

# now set up the public IP, DNS, and certificate
gcloud --project=${PROJECT} compute addresses create mcg-ingress-ip --global

export MCG_INGRESS_IP=$(gcloud --project=${PROJECT} compute addresses describe mcg-ingress-ip --global --format "value(address)")
echo ${MCG_INGRESS_IP}

cat <<EOF > dns-spec.yaml
swagger: "2.0"
info:
  description: "Cloud Endpoints DNS"
  title: "Cloud Endpoints DNS"
  version: "1.0.0"
paths: {}
host: "frontend.endpoints.${PROJECT}.cloud.goog"
x-google-endpoints:
- name: "frontend.endpoints.${PROJECT}.cloud.goog"
  target: "${MCG_INGRESS_IP}"
EOF

gcloud --project=${PROJECT} endpoints services deploy dns-spec.yaml

#create certificate 
gcloud --project=${PROJECT} certificate-manager certificates create mcg-cert \
    --domains="frontend.endpoints.${PROJECT}.cloud.goog"

# create certificate map 
gcloud --project=${PROJECT} certificate-manager maps create mcg-cert-map

# create certificate map entry
gcloud --project=${PROJECT} certificate-manager maps entries create mcg-cert-map-entry \
    --map="mcg-cert-map" \
    --certificates="mcg-cert" \
    --hostname="frontend.endpoints.${PROJECT}.cloud.goog"

# create health check for ingress gateways and cloud armor policies 
gcloud compute security-policies create edge-fw-policy \
    --project=${PROJECT} --description "Block XSS attacks"

gcloud compute security-policies rules create 1000 \
    --security-policy edge-fw-policy \
    --expression "evaluatePreconfiguredExpr('xss-stable')" \
    --action "deny-403" \
    --project=${PROJECT} \
    --description "XSS attack filtering" 

cat <<EOF > cloud-armor-backendpolicy.yaml
apiVersion: networking.gke.io/v1
kind: GCPBackendPolicy
metadata:
  name: cloud-armor-backendpolicy
  namespace: ${IG_NAMESPACE}
spec:
  default:
    securityPolicy: edge-fw-policy
  targetRef:
    group: net.gke.io
    kind: ServiceImport
    name: asm-ingressgateway
EOF

# apply backend policy
kubectl --context=${CLUSTER_1} apply -f cloud-armor-backendpolicy.yaml
kubectl --context=${CLUSTER_2} apply -f cloud-armor-backendpolicy.yaml

cat <<EOF > ingress-gateway-healthcheck.yaml
apiVersion: networking.gke.io/v1
kind: HealthCheckPolicy
metadata:
  name: ingress-gateway-healthcheck
  namespace: ${IG_NAMESPACE}
spec:
  default:
    config:
      httpHealthCheck:
        port: 15021
        portSpecification: USE_FIXED_PORT
        requestPath: /healthz/ready
      type: HTTP
  targetRef:
    group: net.gke.io
    kind: ServiceImport
    name: asm-ingressgateway
EOF

# apply the healthcheck
kubectl --context=${CLUSTER_1} apply -f ingress-gateway-healthcheck.yaml
kubectl --context=${CLUSTER_2} apply -f ingress-gateway-healthcheck.yaml

# set up demo `whereami` app across both clusters
# get app namespaces created
kubectl --context=${CLUSTER_1} create ns frontend
kubectl --context=${CLUSTER_1} label namespace frontend istio-injection=enabled
kubectl --context=${CLUSTER_1} create ns backend
kubectl --context=${CLUSTER_1} label namespace backend istio-injection=enabled
kubectl --context=${CLUSTER_2} create ns frontend
kubectl --context=${CLUSTER_2} label namespace frontend istio-injection=enabled
kubectl --context=${CLUSTER_2} create ns backend
kubectl --context=${CLUSTER_2} label namespace backend istio-injection=enabled

mkdir whereami-backend
mkdir whereami-backend/base

cat <<EOF > whereami-backend/base/kustomization.yaml 
resources:
  - github.com/GoogleCloudPlatform/kubernetes-engine-samples/quickstarts/whereami/k8s
EOF

mkdir whereami-backend/variant

cat <<EOF > whereami-backend/variant/cm-flag.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: whereami
data:
  BACKEND_ENABLED: "False" # assuming you don't want a chain of backend calls
  METADATA:        "backend"
EOF

cat <<EOF > whereami-backend/variant/service-type.yaml 
apiVersion: "v1"
kind: "Service"
metadata:
  name: "whereami"
spec:
  type: ClusterIP
EOF

cat <<EOF > whereami-backend/variant/kustomization.yaml 
nameSuffix: "-backend"
namespace: backend
commonLabels:
  app: whereami-backend
resources:
- ../base
patches:
- path: cm-flag.yaml
  target:
    kind: ConfigMap
- path: service-type.yaml
  target:
    kind: Service
EOF

kubectl --context=${CLUSTER_1} apply -k whereami-backend/variant
kubectl --context=${CLUSTER_2} apply -k whereami-backend/variant

mkdir whereami-frontend
mkdir whereami-frontend/base

cat <<EOF > whereami-frontend/base/kustomization.yaml 
resources:
  - github.com/GoogleCloudPlatform/kubernetes-engine-samples/quickstarts/whereami/k8s
EOF

mkdir whereami-frontend/variant

cat <<EOF > whereami-frontend/variant/cm-flag.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: whereami
data:
  BACKEND_ENABLED: "True"
  BACKEND_SERVICE:        "http://whereami-backend.backend.svc.cluster.local"
EOF

cat <<EOF > whereami-frontend/variant/service-type.yaml 
apiVersion: "v1"
kind: "Service"
metadata:
  name: "whereami"
spec:
  type: ClusterIP
EOF

cat <<EOF > whereami-frontend/variant/kustomization.yaml 
nameSuffix: "-frontend"
namespace: frontend
commonLabels:
  app: whereami-frontend
resources:
- ../base
patches:
- path: cm-flag.yaml
  target:
    kind: ConfigMap
- path: service-type.yaml
  target:
    kind: Service
EOF

kubectl --context=${CLUSTER_1} apply -k whereami-frontend/variant
kubectl --context=${CLUSTER_2} apply -k whereami-frontend/variant

# set up actual frontend load balancing
cat <<EOF > frontend-gateway.yaml 
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: external-http
  namespace: ${IG_NAMESPACE}
  annotations:
    networking.gke.io/certmap: mcg-cert-map
spec:
  gatewayClassName: gke-l7-global-external-managed-mc
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    allowedRoutes:
      kinds:
      - kind: HTTPRoute
  addresses:
  - type: NamedAddress
    value: mcg-ingress-ip
EOF

kubectl --context=${CLUSTER_1} apply -f frontend-gateway.yaml # only need first cluster since it's the config cluster

cat << EOF > default-httproute.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: default-httproute
  namespace: ${IG_NAMESPACE}
spec:
  parentRefs:
  - name: external-http
    namespace: ${IG_NAMESPACE}
    sectionName: https
  rules:
  - backendRefs:
    - group: net.gke.io
      kind: ServiceImport
      name: asm-ingressgateway
      port: 443
EOF

kubectl --context=${CLUSTER_1} apply -f default-httproute.yaml
kubectl --context=${CLUSTER_2} apply -f default-httproute.yaml

# create a virtualService to route requests to the `whereami` frontend
cat << EOF > frontend-vs.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: whereami-vs
  namespace: frontend
spec:
  gateways:
  - ${IG_NAMESPACE}/asm-ingressgateway
  hosts:
  - 'frontend.endpoints.${PROJECT}.cloud.goog'
  http:
  - route:
    - destination:
        host: whereami-frontend
        port:
          number: 80
EOF

kubectl --context=${CLUSTER_1} apply -f frontend-vs.yaml
kubectl --context=${CLUSTER_2} apply -f frontend-vs.yaml

# finally, set up destination rules to keep requests local to the receiving region(s)
cat << EOF > frontend-dr.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: frontend
  namespace: frontend
spec:
  host: whereami-frontend.frontend.svc.cluster.local
  trafficPolicy:
    connectionPool:
      http:
        maxRequestsPerConnection: 0
    loadBalancer:
      simple: LEAST_REQUEST
      localityLbSetting:
        enabled: true
        failover:
          - from: ${REGION_1}
            to: ${REGION_2}
          - from: ${REGION_2}
            to: ${REGION_1}
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 1m
EOF

kubectl --context=${CLUSTER_1} apply -f frontend-dr.yaml
kubectl --context=${CLUSTER_2} apply -f frontend-dr.yaml

cat << EOF > backend-dr.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: backend
  namespace: backend
spec:
  host: whereami-backend.backend.svc.cluster.local
  trafficPolicy:
    connectionPool:
      http:
        maxRequestsPerConnection: 0
    loadBalancer:
      simple: LEAST_REQUEST
      localityLbSetting:
        enabled: true
        failover:
          - from: ${REGION_1}
            to: ${REGION_2}
          - from: ${REGION_2}
            to: ${REGION_1}
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 1m
EOF

kubectl --context=${CLUSTER_1} apply -f backend-dr.yaml
kubectl --context=${CLUSTER_2} apply -f backend-dr.yaml

# test the target
curl https://frontend.endpoints.${PROJECT}.cloud.goog -s | jq "."
```
