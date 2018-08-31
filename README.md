# hasura-graphql-on-gcloud
Hasura GraphQL Engine on Google Cloud

## Pre-requisites

1. Google cloud account with billing enabled
2. `gcloud` CLI

## 1. Create a Google Cloud Project

## 2. Create a Google Cloud SQL Postgres instance

Create a PostgreSQL instance:

```bash
gcloud sql instances create hge-pg --database-version=POSTGRES_9_6 \
       --cpu=1 --memory=3840MiB --region=asia-south1
```

Set a password for `postgres` user:

```bash
gcloud sql users set-password postgres no-host --instance=hge-pg \
       --password=[PASSWORD]
```

Make a note of the `[PASSWORD]`.

## 3. Create a Kubernetes cluster

```bash
gcloud container clusters create hge-k8s \
      --zone asia-south1-a \
      --num-nodes 1
```

## 4. Configure a service account

Create a service account and download the json file by following [this
guide](https://cloud.google.com/sql/docs/postgres/connect-kubernetes-engine#2_create_a_service_account)
or by executing the commands below:

Create a service account:
```bash
gcloud iam service-accounts create hge-sql-sa \
      --display-name hge-sql-sa
```

Grant required roles to this service account (replace `[PROJECT-ID]` with your
project id):
```bash
gcloud projects add-iam-policy-binding [PROJECT-ID] \
    --member serviceAccount:hge-sql-sa@[PROJECT-ID].iam.gserviceaccount.com \
    --role roles/cloudsql.admin
```

Download a json key file:
```json
gcloud iam service-accounts keys create key.json \
  --iam-account hge-sql-sa@[PROJECT-ID].iam.gserviceaccount.com
```

Create a k8s secret with this service account:
```bash
kubectl create secret generic cloudsql-instance-credentials \
    --from-file=credentials.json=key.json
```

Create another secret with the database user and password
(Use the `[PASSWORD]` noted earlier):
```bash
kubectl create secret generic cloudsql-db-credentials \
    --from-literal=username=postgres --from-literal=password=[PASSWORD]
```

## 5. Configure and deploy GraphQL Engine

Get the `INSTANCE_CONNECTION_NAME`:
```bash
gcloud sql instances describe hge-pg
# it'll be something like this:
# connectionName: myproject1:us-central1:myinstance1
```

Edit `deployment.yaml` and replace `INSTANCE_CONNECTION_NAME` with the value.

Create the deployment:

```bash
kubectl create -f deployment.yaml
```

Ensure the pod is running:

```bash
kubectl get pods
```

Check logs if there are errors.

Expose GraphQL Engine on a Google LoadBalancer:

```bash
kubectl expose deploy/hasura-graphql-engine \
        --port 80 --target-port 8080 \
        --type LoadBalancer
```

Get the external IP:
```bash
kubectl get service
```

Open the console by navigating the the external IP in a browser.

Note down this IP as `HGE_IP`.

## 6. Create a sample table

Create the table using the console:

```
Table name: profile

Columns:

id: Integer auto-increment
name: Text
address: Text
lat: Numeric, Nullable
lng: Numeric, Nullable
```

## 7. Create a Google Cloud Function

```bash
gcloud components update && \
gcloud components install beta
```

Choose one the following environments:

- [`nodejs-echo`](cloudfunction/nodejs-echo)
