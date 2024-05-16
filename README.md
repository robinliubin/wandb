# wandb trial on GCP
## prepare installer
- using local machine(macOS) as installer
- ref: https://docs.wandb.ai/guides/hosting/self-managed/gcp-tf
#### install tfenv
```bash
brew install tfenv
```
#### install terraform

##### find out latest terraform version:
```bash
# tfenv list-remote | head -5
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

#### install helm/kubectl
```bash
brew install helm
brew install kubectl
```
