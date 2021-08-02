# KCC SetUp

## Requirements
- kpt
- Docker
- gcloud

0. Set some helpful vars.

```
PROJECT_ID=
SA_NAME=
NAMESPACE=
CLUSTER=
AUTH_NETWORK=
```

1. Create a private cluster to host the ConfigController Specs

```
gcloud beta container --project $PROJECT_ID clusters create $CLUSTER --zone "us-central1-c" \
    --no-enable-basic-auth --cluster-version "1.20.8-gke.900" --release-channel "regular" \
    --machine-type "e2-medium" --image-type "COS_CONTAINERD" --disk-type "pd-standard" --disk-size "100" \
    --metadata disable-legacy-endpoints=true \
    --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" \
    --max-pods-per-node "110" --num-nodes "1" --enable-stackdriver-kubernetes --enable-private-nodes \
    --master-ipv4-cidr "172.16.0.0/28" --enable-master-global-access \
    --enable-ip-alias --network "projects/${PROJECT}/global/networks/default" --subnetwork "projects/${PROJECT}/regions/us-central1/subnetworks/default" \
    --no-enable-intra-node-visibility --default-max-pods-per-node "110" --enable-autoscaling --min-nodes "1" --max-nodes "3" \
    --enable-dataplane-v2 --enable-master-authorized-networks --master-authorized-networks $AUTH_NETWORK \
    --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade \
    --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --workload-pool "${PROJECT_ID}.svc.id.goog" \
    --enable-shielded-nodes --shielded-secure-boot --node-locations "us-central1-c"
```

2. Install the latest version of Config Connector. This is needed for the `GKEHubFeatureMembership` and `GKEHubFeature` resources as they aren't available in stable right now.

```
gsutil cp gs://configconnector-operator/latest/release-bundle.tar.gz release-bundle.tar.gz
tar zxvf release-bundle.tar.gz
kubectl apply -f operator-system/configconnector-operator.yaml
```

2. Create Service Account to use

```
gcloud iam service-accounts create $SA_NAME
```

3. Set role binding

```
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/owner"
```

4. Link the SA account to the Kubernetes SA
```
gcloud iam service-accounts add-iam-policy-binding \
    ${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \
    --member="serviceAccount:${PROJECT_ID}.svc.id.goog[cnrm-system/cnrm-controller-manager]" \
    --role="roles/iam.workloadIdentityUser"
```

5. Configure the Config Connector

```
cat << EOF > configconnector.yaml
apiVersion: core.cnrm.cloud.google.com/v1beta1
kind: ConfigConnector
metadata:
  # the name is restricted to ensure that there is only one
  # ConfigConnector resource installed in your cluster
  name: configconnector.core.cnrm.cloud.google.com
spec:
 mode: cluster
 googleServiceAccount: "${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
EOF
```

Apply the config to the cluster.

`kubectl apply -f configconnector.yaml`


6. Create a namespace for the configs to reside in.

```
kubectl create namespace $NAMESPACE
```

Annotate the namespace with the project id

```
kubectl annotate namespace \
    $NAMESPACE cnrm.cloud.google.com/project-id=${PROJECT_ID}
```

6. Pull the `kpt` package

```
kpt pkg get https://github.com/cartyc/kcc-private-cluster.git
```

6. Edit the `setters.yaml` file to set the required variables. Following that to apply the variables to the files run the below `kpt` function.

```
cd kcc-private-cluster
kpt fn render .
```

Finally apply the configs

```
kubectl apply -f infra/
```