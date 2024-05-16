# wandb trial on GCP
## prepare installer
> https://docs.wandb.ai/guides/hosting/self-managed/gcp-tf
- using local machine(macOS) as installer
- using GCP existing project: `eng-robin-vllm-benchmark`
- pre-exising GCP bucket: `robin-tfstate-gcs-bucket`
- my user has `Owner` role on the project
- already authenticated with gcloud `gcloud auth application-default login`

> ignore if already done:
##### install tfenv
```bash
brew install tfenv
```
##### find out latest terraform version:
```bash
~ tfenv list-remote | head -5
1.9.0-alpha20240501
1.9.0-alpha20240404
1.8.3
1.8.2
1.8.1
```
##### install terraform 1.8.3 not alpha
```bash
tfenv install 1.8.3
tfenv use 1.8.3
```
verify terraform version
```bash
~ terraform version
Terraform v1.8.3
on darwin_arm64
``` 

##### install helm/kubectl
```bash
brew install helm
brew install kubectl
```

## prepare terraform files
#### copy over content from wandb doc for dedicated cloud gcp
https://docs.wandb.ai/guides/hosting/self-managed/gcp-tf
1. Create the terraform.tfvars file.
2. Create the file variables.tf
3. Create the main.tf

4. create a terraform backend file: `backend-gcp.tfvars`
```hcl
bucket  = "robin-tfstate-gcs-bucket"
prefix  = "wandb"
```

## Deploy W&B

To deploy W&B, execute the following commands:
```
terraform init -backend-config=backend-gcp.tfvars
terraform apply -var-file=terraform.tfvars
```

it creates below GCP resources:
- a GKE cluster: `wandb-cluster`
  - n1-standard-4 4vcpu, 15GB RAM, 100GB disk, 2 nodes
- a Cloud SQL instance: `wandb-main-lioness`
- a Memorystore: `wandb-redis`
- a GCS bucket: `wandb-tfstate-gcs-bucket`
## it takes about 10 minutes to deploy

scaling cmd alias:
alias scaleup='gcloud container clusters resize hac-robin-cluster --region=us-central1 --node-pool=gpu --num-nodes=2 -q > /dev/null 2>&1 &

issue1 :
```bash
│ Error: Error creating KeyRing: googleapi: Error 403: Google Cloud KMS API has not been used in project 35775501766 before or it is disabled. Enable it by visiting https://console.developers.google.com/apis/api/cloudkms.googleapis.com/overview?project=35775501766 then retry. If you enabled this API recently, wait a few minutes for the action to propagate to our systems and retry.
│ 
│   with module.wandb.module.kms[0].google_kms_key_ring.default,
│   on .terraform/modules/wandb/modules/kms/main.tf line 6, in resource "google_kms_key_ring" "default":
│    6: resource "google_kms_key_ring" "default" {
│ 
```
fixed by run terraform apply again

issue2:
```bash
╷
│ Error: Error creating Topic: googleapi: Error 400: Cloud Pub/Sub did not have the necessary permissions configured to support this operation. Please verify that the service account service-35775501766@gcp-sa-pubsub.iam.gserviceaccount.com was granted the Cloud KMS CryptoKey Encrypter/Decrypter role for the project containing the CryptoKey resource projects/eng-robin-vllm-benchmark/locations/global/keyRings/wandb-evolving-joey/cryptoKeys/wandb-key.
│ 
│   with module.wandb.module.storage[0].module.pubsub[0].google_pubsub_topic.file_storage,
│   on .terraform/modules/wandb/modules/storage/pubsub/main.tf line 5, in resource "google_pubsub_topic" "file_storage":
│    5: resource "google_pubsub_topic" "file_storage" {
│ 
```
fixed by run terraform apply again

issue3:
```bash
│ Error: Kubernetes cluster unreachable: invalid configuration: no configuration has been provided, try setting KUBERNETES_MASTER environment variable
│ 
│   with module.wandb.module.wandb.helm_release.operator,
│   on .terraform/modules/wandb.wandb/main.tf line 1, in resource "helm_release" "operator":
│    1: resource "helm_release" "operator" {
│ 
```
fixed by connecting:
```bash
gcloud container clusters get-credentials wandb-cluster --zone us-central1-a --project eng-robin-vllm-benchmark
export KUBECONFIG=/Users/binliu/.kube/config
export KUBE_CONFIG_PATH=/Users/binliu/.kube/config
```

untill all terraform apply completed successfully:
```bash
Apply complete! Resources: 0 added, 0 changed, 2 destroyed.

Outputs:

address = "34.49.206.72"
bucket_name = "wandb-happy-grubworm"
url = "https://wandb-robin.gke2.haic.me"
```

## Access W&B
### install nginx ingress controller:
```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```
### create a secret for the TLS cert:
1. create a file `tls-secret.yaml` with public CA signed cert/key:
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: tls-secret
     namespace: default
   data:
     tls.crt: LS0tXXXXXX
     tls.key: LS0tLS1XXXX
   type: kubernetes.io/tls
   ```
2. `kubectl apply -f tls-secret.yaml`
### create ingress for W&B:
1. create a file `wandb-ingress.yaml`:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: wandb-nginx
     namespace: default
   spec:
     ingressClassName: nginx
     rules:
     - host: wandb-robin.gke2.haic.me
       http:
         paths:
         - backend:
             service:
               name: wandb-app
               port:
                 number: 8080
           path: /
           pathType: Prefix
         - backend:
             service:
               name: wandb-console
               port:
                 number: 8082
           path: /console
           pathType: Prefix
     tls:
     - hosts:
       - wandb-gcp-robin.haic.me
       secretName: tls-secret
   ``` 
2. `kubectl apply -f wandb-ingress.yaml`

### config route53 to add a A record for the domain `wandb-robin.gke2.haic.me` to the load balancer IP`

### access the W&B instance at the URL provided in the output
1. browser goto: https://wandb-robin.gke2.haic.me
2. create a testuser
3. note down api-key
   
### using wandb cli:
1. install wandb cli
   ```bash
   pip install wandb
   wandb login --relogin --host=https://wandb-robin.gke2.haic.me
   # copy/paste api-key when prompted
   ```
### run toy project:
1. create test.py with:
   ```python
   import wandb
   import random

   # start a new wandb run to track this script
   wandb.init(
       # set the wandb project where this run will be logged
       project="my-awesome-project",

       # track hyperparameters and run metadata
       config={
       "learning_rate": 0.02,
       "architecture": "CNN",
       "dataset": "CIFAR-100",
       "epochs": 10,
       }
   )

   # simulate training
   epochs = 10
   offset = random.random() / 5
   for epoch in range(2, epochs):
       acc = 1 - 2 ** -epoch - random.random() / epoch - offset
       loss = 2 ** -epoch + random.random() / epoch + offset

       # log metrics to wandb
       wandb.log({"acc": acc, "loss": loss})

   # [optional] finish the wandb run, necessary in notebooks
   wandb.finish()
   ```
### 