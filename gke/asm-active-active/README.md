# [WORK IN PROGRESS] Active-Active Setup with Anthos Service Mesh

## High Level Workflow
The following is the high level workflow which you will follow:
1. Clone this repo
2. Create two Active-Active participating GKE clusters
3. Install Anthos Service Mesh (ASM) and Ingress Gateway
4. Install/Configure Redis Enterprise Operator & Redis Enteprise Cluster
5. ..
6. ..


#### 1. Clone this repo 
```
git clone https://github.com/Redislabs-Solution-Architects/redis-enterprise-cloud-gcp
cd redis-enterprise-asm-ingress/gke/asm-active-active
```
    
#### 2. Create two Active-Active participating GKE clusters
Create first GKE cluster:
```
export PROJECT_ID=$(gcloud info --format='value(config.project)')
export CLUSTER_LOCATION_01=us-central1
export CLUSTER_NAME_01="glau-gke-cluster-$CLUSTER_LOCATION_01"
export VPC_NETWORK="glau-vpc-network"
export SUBNETWORK_01="us-central1-subnet"

gcloud container clusters create $CLUSTER_NAME_01 \
 --region=$CLUSTER_LOCATION_01 --num-nodes=1 \
 --image-type=COS_CONTAINERD \
 --machine-type=e2-standard-8 \
 --network=$VPC_NETWORK \
 --subnetwork=$SUBNETWORK_01 \
 --enable-ip-alias \
 --workload-pool=${PROJECT_ID}.svc.id.goog
```

Create second GKE cluster:
```
export CLUSTER_LOCATION_02=us-west1
export CLUSTER_NAME_02="glau-gke-cluster-$CLUSTER_LOCATION_02"
export SUBNETWORK_02="us-west1-subnet"

gcloud container clusters create $CLUSTER_NAME_02 \
 --region=$CLUSTER_LOCATION_02 --num-nodes=1 \
 --image-type=COS_CONTAINERD \
 --machine-type=e2-standard-8 \
 --network=$VPC_NETWORK \
 --subnetwork=$SUBNETWORK_02 \
 --enable-ip-alias \
 --workload-pool=${PROJECT_ID}.svc.id.goog
```
    
#### 3. Install Anthos Service Mesh (ASM) and Ingress Gateway
Download ASM install script:
```
curl https://storage.googleapis.com/csm-artifacts/asm/asmcli_1.16 > asmcli
chmod +x asmcli
```

Set up first GKE cluster:
```
gcloud container clusters get-credentials $CLUSTER_NAME_01 --region $CLUSTER_LOCATION_01 --project $PROJECT_ID
```
Run `asmcli validate` to make sure that your project and cluster are set up as required to install Anthos Service Mesh:
```
./asmcli validate \
  --project_id $PROJECT_ID \
  --cluster_name $CLUSTER_NAME_01 \
  --cluster_location $CLUSTER_LOCATION_01 \
  --fleet_id $PROJECT_ID \
  --output_dir ./asm_output_01
```
Install Anthos Service Mesh:
```
./asmcli install \
  --project_id $PROJECT_ID\
  --cluster_name $CLUSTER_NAME_01 \
  --cluster_location $CLUSTER_LOCATION_01\
  --output_dir ./asm_output_01 \
  --enable_all \
  --ca mesh_ca
  ```
On success, you should have output similar to the following:
```
asmcli: ...done!
asmcli:
asmcli: *****************************
client version: 1.16.2-asm.2
control plane version: 1.16.2
data plane version: none
asmcli: *****************************
asmcli: The ASM control plane installation is now complete.
asmcli: To enable automatic sidecar injection on a namespace, you can use the following command:
asmcli: kubectl label namespace <NAMESPACE> istio-injection- istio.io/rev=asm-1162-2 --overwrite
asmcli: If you use 'istioctl install' afterwards to modify this installation, you will need
asmcli: to specify the option '--set revision=asm-1162-2' to target this control plane
asmcli: instead of installing a new one.
asmcli: To finish the installation, enable Istio sidecar injection and restart your workloads.
asmcli: For more information, see:
asmcli: https://cloud.google.com/service-mesh/docs/proxy-injection
asmcli: The ASM package used for installation can be found at:
asmcli: /home/gilbert_lau/work/asm_output_01/asm
asmcli: The version of istioctl that matches the installation can be found at:
asmcli: /home/gilbert_lau/work/asm_output_01/istio-1.16.2-asm.2/bin/istioctl
asmcli: A symlink to the istioctl binary can be found at:
asmcli: /home/gilbert_lau/work/asm_output_01/istioctl
asmcli: The combined configuration generated for installation can be found at:
asmcli: /home/gilbert_lau/work/asm_output_01/asm-1162-2-manifest-raw.yaml
asmcli: The full, expanded set of kubernetes resources can be found at:
asmcli: /home/gilbert_lau/work/asm_output_01/asm-1162-2-manifest-expanded.yaml
asmcli: *****************************
asmcli: Successfully installed ASM.
```
    
Install Istio Ingress Gateway:
```
kubectl config set-context --current --namespace=istio-system
export asm_version=$(kubectl get deploy -n istio-system -l app=istiod \
  -o=jsonpath='{.items[*].metadata.labels.istio\.io\/rev}''{"\n"}')
kubectl label namespace istio-system istio-injection=enabled istio.io/rev=$asm_version --overwrite
kubectl apply -f ./asm_output_01/samples/gateways/istio-ingressgateway
kubectl apply -f ./asm_output_01/samples/gateways/istio-ingressgateway/autoscalingv2/autoscaling-v2.yaml
```
    
Create a Private DNS zone in your Google Cloud envrionment:
```
export DNS_ZONE=private-redis-zone
export DNS_SUFFIX=istio.k8s.redis.com
gcloud dns managed-zones create $DNS_ZONE \
    --description=$DNS_ZONE \
    --dns-name=$DNS_SUFFIX \
    --networks=$VPC_NETWORK \
    --labels=project=$PROJECT_ID \
    --visibility=private
```
Create DNS entry in your Google Cloud environment:
```
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway \
       -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
gcloud dns record-sets create *.${CLUSTER_LOCATION_01}.${DNS_SUFFIX}. \
    --type=A --ttl=300 --rrdatas=${INGRESS_HOST} --zone=$DNS_ZONE 
```
    
Set up second GKE cluster:
```
gcloud container clusters get-credentials $CLUSTER_NAME_02 --region $CLUSTER_LOCATION_02 --project $PROJECT_ID
```
Run `asmcli validate` to make sure that your project and cluster are set up as required to install Anthos Service Mesh:
```
./asmcli validate \
  --project_id $PROJECT_ID \
  --cluster_name $CLUSTER_NAME_02 \
  --cluster_location $CLUSTER_LOCATION_02 \
  --fleet_id $PROJECT_ID \
  --output_dir ./asm_output_02
```
Install Anthos Service Mesh:
```
./asmcli install \
  --project_id $PROJECT_ID\
  --cluster_name $CLUSTER_NAME_02 \
  --cluster_location $CLUSTER_LOCATION_02\
  --output_dir ./asm_output_02 \
  --enable_all \
  --ca mesh_ca
  ```
    
Install Istio Ingress Gateway:
```
kubectl config set-context --current --namespace=istio-system
export asm_version=$(kubectl get deploy -n istio-system -l app=istiod \
  -o=jsonpath='{.items[*].metadata.labels.istio\.io\/rev}''{"\n"}')
kubectl label namespace istio-system istio-injection=enabled istio.io/rev=$asm_version --overwrite
kubectl apply -f ./asm_output_02/samples/gateways/istio-ingressgateway
kubectl apply -f ./asm_output_02/samples/gateways/istio-ingressgateway/autoscalingv2/autoscaling-v2.yaml
```
    
Create DNS entry in your Google Cloud environment:
```
export DNS_ZONE=private-redis-zone
export DNS_SUFFIX=istio.k8s.redis.com
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway \
       -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
gcloud dns record-sets create *.${CLUSTER_LOCATION_02}.${DNS_SUFFIX}. \
    --type=A --ttl=300 --rrdatas=${INGRESS_HOST} --zone=$DNS_ZONE 
```
    
#### 4. Install/Configure Redis Enterprise Operator & Redis Enteprise Cluster
Install Redis Enterprise Operator & Redis Enterprise Cluster on first GKE cluster:
```
gcloud container clusters get-credentials $CLUSTER_NAME_01 --region $CLUSTER_LOCATION_01 --project $PROJECT_ID

kubectl create namespace $CLUSTER_LOCATION_01
kubectl config set-context --current --namespace=$CLUSTER_LOCATION_01

VERSION=`curl --silent https://api.github.com/repos/RedisLabs/redis-enterprise-k8s-docs/releases/latest | grep tag_name | awk -F'"' '{print $4}'`
kubectl apply -f https://raw.githubusercontent.com/RedisLabs/redis-enterprise-k8s-docs/$VERSION/bundle.yaml

kubectl apply -f - <<EOF
apiVersion: "app.redislabs.com/v1"
kind: "RedisEnterpriseCluster"
metadata:
  name: redis-enterprise
spec:
  nodes: 3
EOF
```
Enable Active-Active controllers:
```
kubectl apply -f https://raw.githubusercontent.com/RedisLabs/redis-enterprise-k8s-docs/master/crds/reaadb_crd.yaml
kubectl apply -f https://raw.githubusercontent.com/RedisLabs/redis-enterprise-k8s-docs/master/crds/rerc_crd.yaml
kubectl patch cm  operator-environment-config --type merge --patch "{\"data\": \
    {\"ACTIVE_ACTIVE_DATABASE_CONTROLLER_ENABLED\":\"true\", \
    \"REMOTE_CLUSTER_CONTROLLER_ENABLED\":\"true\"}}"
```
    
Configure Istio for Redis Enterprise Kubernetes operator to allow external access to Redis Enterprise databases:
```
export DNS_SUFFIX=istio.k8s.redis.com

kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: redis-gateway
spec:
  selector:
    istio: ingressgateway 
  servers:
  - hosts:
    - '*.${CLUSTER_LOCATION_01}.${DNS_SUFFIX}'
    port:
      name: https
      number: 443
      protocol: HTTPS
    tls:
      mode: PASSTHROUGH
EOF
```
```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: redis-vs
spec:
  gateways:
  - redis-gateway
  hosts:
  - '*.${CLUSTER_LOCATION_01}.${DNS_SUFFIX}'
  tls:
  - match:
    - port: 443
      sniHosts:
      - api.${CLUSTER_LOCATION_01}.${DNS_SUFFIX}
    route:
    - destination:
        host: redis-enterprise
        port:
          number: 9443
  - match:
    - port: 443
      sniHosts:
      - redis-enterprise-ui.${CLUSTER_LOCATION_01}.${DNS_SUFFIX}
    route:
    - destination:
        host: redis-enterprise-ui
        port:
          number: 8443
EOF
```
Verify Redis Enterprise endpoints are accessible through gateway:
```
kubectl describe svc istio-ingressgateway -n istio-system
```
Make sure Endpoints lines are not empty from the output:
```
Name:                     istio-ingressgateway
Namespace:                istio-system
Labels:                   app=istio-ingressgateway
                          istio=ingressgateway
Annotations:              cloud.google.com/neg: {"ingress":true}
Selector:                 app=istio-ingressgateway,istio=ingressgateway
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.88.4.232
IPs:                      10.88.4.232
LoadBalancer Ingress:     34.71.227.19
Port:                     status-port  15021/TCP
TargetPort:               15021/TCP
NodePort:                 status-port  31523/TCP
Endpoints:                10.84.0.6:15021,10.84.1.10:15021,10.84.2.5:15021
Port:                     http2  80/TCP
TargetPort:               80/TCP
NodePort:                 http2  30237/TCP
Endpoints:                10.84.0.6:80,10.84.1.10:80,10.84.2.5:80
Port:                     https  443/TCP
TargetPort:               443/TCP
NodePort:                 https  31669/TCP
Endpoints:                10.84.0.6:443,10.84.1.10:443,10.84.2.5:443
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason                Age   From                Message
  ----    ------                ----  ----                -------
  Normal  EnsuringLoadBalancer  17m   service-controller  Ensuring load balancer
  Normal  EnsuredLoadBalancer   17m   service-controller  Ensured load balancer
```
Create RedisEnterpriseRemoteCluster resources:
```
kubectl apply -f - <<EOF
apiVersion: app.redislabs.com/v1alpha1
kind: RedisEnterpriseRemoteCluster
metadata:
  name: rerc-$CLUSTER_NAME_01
spec:
  recName: redis-enterprise
  recNamespace: $CLUSTER_LOCATION_01
  apiFqdnUrl: api.${CLUSTER_LOCATION_01}.${DNS_SUFFIX}
  dbFqdnSuffix: -db.${CLUSTER_LOCATION_01}.${DNS_SUFFIX}
  secretName: redis-enterprise
EOF
```

              
Install Redis Enterprise Operator & Redis Enterprise Cluster on second GKE cluster:
```
gcloud container clusters get-credentials $CLUSTER_NAME_02 --region $CLUSTER_LOCATION_02 --project $PROJECT_ID

kubectl create namespace $CLUSTER_LOCATION_02
kubectl config set-context --current --namespace=$CLUSTER_LOCATION_02

VERSION=`curl --silent https://api.github.com/repos/RedisLabs/redis-enterprise-k8s-docs/releases/latest | grep tag_name | awk -F'"' '{print $4}'`
kubectl apply -f https://raw.githubusercontent.com/RedisLabs/redis-enterprise-k8s-docs/$VERSION/bundle.yaml

kubectl apply -f - <<EOF
apiVersion: "app.redislabs.com/v1"
kind: "RedisEnterpriseCluster"
metadata:
  name: redis-enterprise
spec:
  nodes: 3
EOF
```
Enable Active-Active controllers:
```
kubectl apply -f https://raw.githubusercontent.com/RedisLabs/redis-enterprise-k8s-docs/master/crds/reaadb_crd.yaml
kubectl apply -f https://raw.githubusercontent.com/RedisLabs/redis-enterprise-k8s-docs/master/crds/rerc_crd.yaml
kubectl patch cm  operator-environment-config --type merge --patch "{\"data\": \
    {\"ACTIVE_ACTIVE_DATABASE_CONTROLLER_ENABLED\":\"true\", \
    \"REMOTE_CLUSTER_CONTROLLER_ENABLED\":\"true\"}}"
```
    
Configure Istio for Redis Enterprise Kubernetes operator to allow external access to Redis Enterprise databases:
```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: redis-gateway
spec:
  selector:
    istio: ingressgateway 
  servers:
  - hosts:
    - '*.${CLUSTER_LOCATION_02}.${DNS_SUFFIX}'
    port:
      name: https
      number: 443
      protocol: HTTPS
    tls:
      mode: PASSTHROUGH
EOF
```
```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: redis-vs
spec:
  gateways:
  - redis-gateway
  hosts:
  - '*.${CLUSTER_LOCATION_02}.${DNS_SUFFIX}'
  tls:
  - match:
    - port: 443
      sniHosts:
      - api.${CLUSTER_LOCATION_02}.${DNS_SUFFIX}
    route:
    - destination:
        host: redis-enterprise
        port:
          number: 9443
  - match:
    - port: 443
      sniHosts:
      - redis-enterprise-ui.${CLUSTER_LOCATION_02}.${DNS_SUFFIX}
    route:
    - destination:
        host: redis-enterprise-ui
        port:
          number: 8443
EOF
```
Verify Redis Enterprise endpoints are accessible through gateway:
```
kubectl describe svc istio-ingressgateway -n istio-system
```
Create RedisEnterpriseRemoteCluster resources:
```
kubectl apply -f - <<EOF
apiVersion: app.redislabs.com/v1alpha1
kind: RedisEnterpriseRemoteCluster
metadata:
  name: rerc-$CLUSTER_NAME_02
spec:
  recName: redis-enterprise
  recNamespace: $CLUSTER_LOCATION_02
  apiFqdnUrl: api.${CLUSTER_LOCATION_02}.${DNS_SUFFIX}
  dbFqdnSuffix: -db.${CLUSTER_LOCATION_02}.${DNS_SUFFIX}
  secretName: redis-enterprise
EOF
```


#### 5. Create Active-Active database (REAADB) 
https://docs.redis.com/latest/kubernetes/active-active/preview/create-reaadb/ 