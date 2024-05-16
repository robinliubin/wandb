# wandb trial on GCP
## prepare installer
- ref: https://docs.wandb.ai/guides/hosting/self-managed/gcp-tf
- using local machine(macOS) as installer
- using GCP existing project: `eng-robin-vllm-benchmark`
- my user has `Owner` role on the project
- already authenticated with gcloud `gcloud auth application-default login`

##### ignore if already done:
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
terraform init
terraform apply -var-file=terraform.tfvars
```