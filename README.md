# KCC SetUP

1. Create a private cluster to host the ConfigController Specs

```
gcloud beta container --project "okd-anthos-demo" clusters create "kcc-master" --zone "us-central1-c" \
    --no-enable-basic-auth --cluster-version "1.20.8-gke.900" --release-channel "regular" \
    --machine-type "e2-medium" --image-type "COS_CONTAINERD" --disk-type "pd-standard" --disk-size "100" \
    --metadata disable-legacy-endpoints=true \
    --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" \
    --max-pods-per-node "110" --num-nodes "1" --enable-stackdriver-kubernetes --enable-private-nodes \
    --master-ipv4-cidr "172.16.0.0/28" --enable-master-global-access \
    --enable-ip-alias --network "projects/okd-anthos-demo/global/networks/default" --subnetwork "projects/okd-anthos-demo/regions/us-central1/subnetworks/default" \
    --no-enable-intra-node-visibility --default-max-pods-per-node "110" --enable-autoscaling --min-nodes "1" --max-nodes "3" \
    --enable-dataplane-v2 --enable-master-authorized-networks --master-authorized-networks 24.52.218.194/32 \
    --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver,ConfigConnector --enable-autoupgrade \
    --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --workload-pool "okd-anthos-demo.svc.id.goog" \
    --enable-shielded-nodes --shielded-secure-boot --tags "kcc-master" --node-locations "us-central1-c"
```

```
kpt fn render
```

Apply the configs

```
kubectl apply -f infra/
```