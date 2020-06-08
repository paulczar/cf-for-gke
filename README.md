# Deploy cf4k8s + extras on GCP

## Pre-reqs

Install k14s via curl|bash:

```bash
curl -s -L https://k14s.io/install.sh | \
  K14SIO_INSTALL_BIN_DIR=~/bin bash
```

Install bosh cli:

*If running on Mac, replace `linux` with `darwin` in the command*

```bash
wget -O ~/bin/bosh https://github.com/cloudfoundry/bosh-cli/releases/download/v6.2.1/bosh-cli-6.2.1-linux-amd64
chmod +x ~/bin/bosh

## Prepare Local Environment

Clone cf4k8s-extras:

```
git clone ...
```

Clone cf4k8s:

```bash
git clone https://github.com/cloudfoundry/cf-for-k8s.git
```

Create a spot for customizations/values for a `dev` instance:

```bash
mkdir -p cf4k8s-foundations/dev
```

## Prepare Google Cloud Account

### Create GKE Cluster

*Ensure that you have enabled the [Cloud IAM Service Account Credentials API](https://console.cloud.google.com/apis/api/iamcredentials.googleapis.com/overview?_ga=2.231642838.851680512.1591144110-657035042.1588876317).*

Create a GKE Cluster appropriately sized for a small cf-for-k8s foundation.

```bash
PROJECT_ID=$(gcloud config get-value project)
gcloud container clusters create cf4k8s-dev \
  --num-nodes=5 --zone us-central1-c \
  --cluster-version 1.16 --machine-type n1-standard-2 \
  --workload-pool=${PROJECT_ID}.svc.id.goog \
  --enable-ip-alias
```

Once the cluster is created check your access:

```bash
$ kubectl cluster-info
Kubernetes master is running at https://...
GLBCDefaultBackend is running at https://.../api/v1/namespaces/kube-system/services/default-http-backend:http/proxy
KubeDNS is running at https://.../api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://.../api/v1/namespaces/kube-system/services/https:metrics-server:/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

### Set up a network peer and service connector

**This only needs to be done once per network per project**

1. Create a VPC Peering range:

```bash
gcloud compute addresses create cloudsql-peer \
    --global \
    --purpose=VPC_PEERING \
    --prefix-length=16 \
    --description="peering range for CloudSQL" \
    --network=default \
    --project=$PROJECT_ID
```

2. Peer that range with our default network:

```bash
gcloud services vpc-peerings connect \
    --service=servicenetworking.googleapis.com \
    --ranges=cloudsql-peer \
    --network=default \
    --project=$PROJECT_ID
```

### Create a static IP

Create a Static IP to use for the foundation:

```bash
gcloud compute addresses create cf4k8s-dev --region us-central1
```

### Install Google Config Connector

* Ensure that you have enabled the ]Cloud IAM Service Account Credentials API](https://console.cloud.google.com/apis/api/iamcredentials.googleapis.com/overview?_ga=2.231642838.851680512.1591144110-657035042.1588876317).

Create a service account for GCC:

```bash
gcloud iam service-accounts create cnrm-system

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:cnrm-system@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/owner"

gcloud iam service-accounts add-iam-policy-binding \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:${PROJECT_ID}.svc.id.goog[cnrm-system/cnrm-controller-manager]" \
cnrm-system@${PROJECT_ID}.iam.gserviceaccount.com
```

Create a config for your environment:

```bash
cp cf4k8s-extras/ytt/values.yaml cf4k8s-foundations/dev/
```

**Edit the copy of this file to suit your environment**

Create a deployment manifest for GCC using `ytt`:

```bash
ytt -f cf4k8s-extras/ytt/gcc \
  -f cf4k8s-foundations/dev/values.yaml \
   > cf4k8s-foundations/dev/google-config-connector.yaml
```

Deploy GCC using `kapp`:

```bash
kapp deploy -a gcc -y \
  -f cf4k8s-foundations/dev/google-config-connector.yaml
```

Validate that GCC is running:

```bash
kubectl wait -n cnrm-system --for=condition=Ready pod --all

```

### Install Infrastructure

Crete Infrastructure Manifests:

```
ytt -f cf4k8s-extras/ytt/infrastructure \
  -f cf4k8s-foundations/dev/values.yaml \
   > cf4k8s-foundations/dev/infrastructure.yaml
```

Deploy Infrastructure using `kapp`:

```bash
kapp deploy -a infrastructure -y \
  -f cf4k8s-foundations/dev/infrastructure.yaml
```

Wait for it to be ready:

```bash
kubectl wait -n infrastructure --for=condition=Ready \
  computeaddresses/cf4k8s-dev-ingress \
  iamserviceaccountkeys/cf4k8s-dev-gcr-access
```

Check the database is working:

```
$ kubectl run -n cf-db -ti --restart=Never --image postgres:13-alpine --rm psql -- sh

$ psql -h cf-db-postgresql -U postgres
password for user postgres: ****
psql (13beta1, server 9.6.16)
Type "help" for help.

postgres=> 
```

## Deploy cf-for-k8s

Get the IP address for our external IP:

```bash
IP=$(kubectl -n infrastructure get \
  computeaddresses.compute.cnrm.cloud.google.com \
  cf4k8s-dev-ingress -o json | jq -r .spec.address)
echo $IP
```

Generate cf-values:

```bash
./cf-for-k8s/hack/generate-values.sh -d "$IP.xip.io" > ./cf4k8s-foundations/dev/cf-values.yml
```

Append cf-values with our static IP and foundation Name:

```bash
echo istio_static_ip: $IP >> ./cf4k8s-foundations/dev/cf-values.yml
foundation=cf4k8s-dev
echo foundationName: $foundation >> ./cf4k8s-foundations/dev/cf-values.yml
```

Create a file `./cf4k8s-foundations/dev/gcp-values.yml`:

```bash
cat <<EOF > ./cf4k8s-foundations/dev/gcp-values.yml
#@data/values
---
app_registry:
  hostname: gcr.io
  repository: gcr.io/${PROJECT_ID}/cf-workloads
  username: _json_key
  password: |
    $(kubectl -n infrastructure get iamserviceaccountkeys.iam.cnrm.cloud.google.com cf4k8s-dev-gcr-access  -o json | jq -r .status.privateKey | base64 --decode | jq -c)

capi:
  blobstore:
    package_directory_key: ${foundation}-cc-packages
    droplet_directory_key: ${foundation}-cc-droplets
    resource_directory_key: ${foundation}-cc-resources
    buildpack_directory_key: ${foundation}-cc-buildpacks
    provider: Google
    google_project: ${PROJECT_ID}
    google_client_email: $(kubectl -n infrastructure get iamserviceaccountkeys.iam.cnrm.cloud.google.com cf4k8s-dev-gcr-access  -o json | jq -r .status.privateKey | base64 --decode | jq -r .client_email)
    google_json_key_string: |
      $(kubectl -n infrastructure get iamserviceaccountkeys.iam.cnrm.cloud.google.com cf4k8s-dev-gcr-access  -o json | jq -r .status.privateKey | base64 --decode | jq -c)

EOF
```


Use `ytt` to render our final manifest for CF:

```bash
ytt -f ./cf-for-k8s/config \
    -f ./cf4k8s-extras/ytt/cf-for-k8s \
    -f ./cf4k8s-foundations/dev/cf-values.yml \
    -f ./cf4k8s-foundations/dev/gcp-values.yml \
     > ./cf4k8s-foundations/dev/cf-for-k8s-rendered.yml
```

Use `kapp` to deploy CF for k8s:

```bash
kapp deploy -y -a cf -f ./cf4k8s-foundations/dev/cf-for-k8s-rendered.yml
```

## Validate Deployment

Log in and create an org/space:

```bash
cf api --skip-ssl-validation https://api.$IP.xip.io
cf auth admin $(yq r cf4k8s-foundations/dev/cf-values.yml cf_admin_password)
cf create-org test-org
cf create-space -o test-org test-space
cf target -o test-org -s test-space
```

Push a test app:

```bash
cf push test-node-app -p ./cf-for-k8s/tests/smoke/assets/test-node-app
```

Test the app:

```bash
curl test-node-app.apps.$IP.xip.io
```

You should see:

```
Hello World
```
