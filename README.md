# sops-hacking

```shell
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
curl -s https://fluxcd.io/install.sh | sudo bash
sudo chmod +x /usr/local/bin/flux
sudo curl -L https://github.com/mozilla/sops/releases/download/v3.7.2/sops-v3.7.2.linux.amd64 -o /usr/local/bin/sops
sudo chmod +x /usr/local/bin/sops

k3d cluster create

export GITHUB_TOKEN=<REDACTED>
flux bootstrap github --owner=noelbundick-msft --repository=sops-hacking --personal=true --path=clusters/codespace --branch=main

export KEY_NAME="codespace.noelbundick.com"
export KEY_COMMENT="flux secrets"

gpg --batch --full-generate-key <<EOF
%no-protection
Key-Type: 1
Key-Length: 4096
Subkey-Type: 1
Subkey-Length: 4096
Expire-Date: 0
Name-Comment: ${KEY_COMMENT}
Name-Real: ${KEY_NAME}
EOF

export SOPS_PGP_FP=81567F76FE58B7C41D855243130D49C0AEF1B869

gpg --export-secret-keys --armor "${SOPS_PGP_FP}" |
kubectl create secret generic sops-gpg \
--namespace=flux-system \
--from-file=sops.asc=/dev/stdin

# Put secrets in Key Vault and delete secrets if you don't need em
#gpg --delete-secret-keys "${SOPS_PGP_FP}"

flux create kustomization my-secrets \
  --source=flux-system \
  --path=apps/clusters/codespace \
  --prune=true \
  --interval=10m \
  --decryption-provider=sops \
  --decryption-secret=sops-gpg

az ad sp create-for-rbac -n noel-sops-hacking
# {
#   "appId": "7ed0bade-c55a-4ad8-81a8-5c581dc1023d",
#   "displayName": "noel-sops-hacking",
#   "password": "<REDACTED>",
#   "tenant": "72f988bf-86f1-41af-91ab-2d7cd011db47"
# }

az group create -n sops-hacking -l westus3
az acr create -g sops-hacking -n sopshacking --sku Standard

# add a role assignment

docker pull nginx
docker tag nginx sopshacking.azurecr.io/nginx
docker push sopshacking.azurecr.io/nginx

kubectl create secret docker-registry acr-secret \
    --namespace nginx \
    --docker-server=sopshacking.azurecr.io \
    --docker-username=7ed0bade-c55a-4ad8-81a8-5c581dc1023d \
    --docker-password=<REDACTED> \
    --dry-run=client -o yaml > ./apps/clusters/codespace/secret.yaml
```
