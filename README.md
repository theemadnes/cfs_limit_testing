# cfs_limit_testing
Run through some tests on latency impact when enabling K8s limits to pods on both single &amp; multithreaded apps, and analyze impact of manipulating CFS period




#### setup cluster & ASM

start with one cluster. going to enable ASM (Istio) to capture latency metrics. 

*note* - i'm installing this from MacOS, so some of the commands (for exampl, getting the ASM & istioctl binaries) will change if you're using something else 

```
# set up environment vars  
export PROJECT=$(gcloud config get-value project) # or your preferred project

export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT} --format="value(projectNumber)")

export CLUSTER_NAME=cfs-limit-test-01 # also going to act as config cluster
export REGION=us-central1
export ASM_VERSION=1.6.8-asm.9

gcloud services enable \
    container.googleapis.com \
    compute.googleapis.com \
    monitoring.googleapis.com \
    logging.googleapis.com \
    cloudtrace.googleapis.com \
    meshca.googleapis.com \
    meshtelemetry.googleapis.com \
    meshconfig.googleapis.com \
    iamcredentials.googleapis.com \
    anthos.googleapis.com \
    cloudresourcemanager.googleapis.com \
    gkehub.googleapis.com \
    multiclusteringress.googleapis.com \
    gkeconnect.googleapis.com


curl -LO https://storage.googleapis.com/gke-release/asm/istio-${ASM_VERSION}-osx.tar.gz

tar xzf istio-${ASM_VERSION}-osx.tar.gz

cd istio-1.6.8-asm.9

export PATH=$PWD/bin:$PATH

istioctl version --remote=false

# i've already downloaded kpt, but you might have to 
kpt pkg get \
https://github.com/GoogleCloudPlatform/anthos-service-mesh-packages.git/asm@release-1.6-asm ${PWD}/asm

curl --request POST \
  --header "Authorization: Bearer $(gcloud auth print-access-token)" \
  --data '' \
"https://meshconfig.googleapis.com/v1alpha1/projects/${PROJECT}:initialize"


# create GKE cluster 

gcloud beta container clusters create ${CLUSTER_NAME} \
    --machine-type=e2-standard-4 \
    --num-nodes=1 \
    --region ${REGION} \
    --enable-stackdriver-kubernetes \
    --enable-ip-alias \
    --workload-pool=${PROJECT}.svc.id.goog \
    --labels=mesh_id=proj-${PROJECT_NUMBER} \
    --release-channel rapid

kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole=cluster-admin \
    --user="$(gcloud config get-value core/account)"

kpt cfg set ${PWD}/asm gcloud.container.cluster ${CLUSTER_NAME}
kpt cfg set ${PWD}/asm gcloud.core.project ${PROJECT}
kpt cfg set ${PWD}/asm gcloud.compute.location ${REGION}
istioctl install -f ${PWD}/asm/cluster/istio-operator.yaml --set values.gateways.istio-ingressgateway.type=ClusterIP

# configure ingress gateway + GCLB integration


cd .. # go back to the root of the project

# enable healthchecks on the pod endpoints 

gcloud compute firewall-rules create fw-allow-health-checks \
    --action=ALLOW \
    --direction=INGRESS \
    --source-ranges=35.191.0.0/16,130.211.0.0/22 \
    --rules=tcp:15021



# apply ingress configuration 

kubectl patch svc istio-ingressgateway -n istio-system --patch "$(cat ingress-gateway-patch/ingress-service-patch.yaml)"

kubectl apply -f ingress-cnlb/
```

#### build golang hello world image 

Going to start with [this](https://yourbasic.org/golang/http-server-example/) example.