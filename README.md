# KCC SetUp

## Requirements
- [kpt](https://kpt.dev/installation/)
- [Docker](https://docs.docker.com/desktop/)
- [gcloud](https://cloud.google.com/sdk/docs/install)

0. Set some helpful vars.

```
CONFIG_CONTROLLER_NAME=kcc-control
```

1. SetUp Config Controller. Full setup can be found [here](https://cloud.google.com/anthos-config-management/docs/how-to/config-controller-setup)

```
gcloud services enable krmapihosting.googleapis.com \
    container.googleapis.com \
    cloudresourcemanager.googleapis.com
    
gcloud alpha anthos config controller create $CONFIG_CONTROLLER_NAME \
    --location=us-central1    
```

2. Switch to the config controller context

```
gcloud alpha anthos config controller get-credentials $CONFIG_CONTROLLER_NAME \
    --location us-central1
```
3. Give Config Controller Access to create resources.
```
export PROJECT_ID=$(gcloud config get-value project)
export SA_EMAIL="$(kubectl get ConfigConnectorContext -n config-control \
    -o jsonpath='{.items[0].spec.googleServiceAccount}' 2> /dev/null)"
gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
    --member "serviceAccount:${SA_EMAIL}" \
    --role "roles/owner" \
    --project "${PROJECT_ID}"
```    

4. Pull the `kpt` package

```
kpt pkg get https://github.com/cartyc/kcc-private-cluster.git
```

5. Edit the `setters.yaml` file to set the required variables. Following that to apply the variables to the files run the below `kpt` function.

```
cd kcc-private-cluster
kpt fn render .
```

Finally apply the configs

```
kubectl apply -f infra/
```
