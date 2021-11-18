# DEMOS


# DEMO 01

Make a new folder and open in container

``` bash
mkdir sample01
cd sample01
code .
```

in Code "open in container"


# DEMO 02

Open Typescript project

# DEMO 03

Create python container

Replace user in dockerfile:

``` dockerfile
# replace vscode user 
ARG USERNAME=john
ARG VSC_CODE_NAME=vscode
RUN usermod -m -d /home/$USERNAME $VSC_CODE_NAME
RUN groupmod -n $USERNAME $VSC_CODE_NAME
RUN usermod -l $USERNAME $VSC_CODE_NAME

# Finish setting up $USERNAME (sudo, no password etc ...)
RUN echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME \
    && chmod 0440 /etc/sudoers.d/$USERNAME 
```

``` json
	"build": {
		"dockerfile": "Dockerfile",
		"context": "..",
		"args": { 
			// Update 'VARIANT' to pick a Python version: 3, 3.10, 3.9, 3.8, 3.7, 3.6
			// Append -bullseye or -buster to pin to an OS version.
			// Use -bullseye variants on local on arm64/Apple Silicon.
			"VARIANT": "3.10-bullseye",
			// Options
			"NODE_VERSION": "none",
			"USERNAME": "ionut"
		}
	},
	// Comment out connect as root instead. More info: https://aka.ms/vscode-remote/containers/non-root.
	"remoteUser": "ionut"
```    

Add env file

``` ini
TENANT_ID=72f988bf-86f1-41af-91ab-2d7cd011db47
SUBSCRIPTION_ID=6d854ccd-e85d-4a3e-baee-af4f95a56211
DOMAIN=microsoft.onmicrosoft.com
AUTHORITY=https://login.microsoftonline.com/72f988bf-86f1-41af-91ab-2d7cd011db47

# py-web
# VAULT_NAME=kvpy
# VAULT_SECRET_KEY=clientsecret
CLIENT_ID=36d717f8-4f97-4ddf-abbf-e7d0ab6a9821

# Scopes
SCOPES=https://microsoft.onmicrosoft.com/b7826cf1-3f0d-4d5a-be06-3c5bfea19fbc/user_impersonation

# py-api
API_CLIENT_ID=b7826cf1-3f0d-4d5a-be06-3c5bfea19fbc
API_CLIENT_SECRET=9rn7Q~nFSgYiELxGqSoKI4n1TWSW51GX12Cyn
```

Ajouter l'extension ENV

``` json
	// Add the IDs of extensions you want installed when the container is created.
	"extensions": [
		"ms-python.python",
		"ms-python.vscode-pylance",
		"irongeek.vscode-env"
	],
```

COPY files from solution inside folder and see nothing work

Install some basics stuff

``` dockerfile
# Install some basics packages
RUN apt-get update && \ 
    apt-get upgrade -y && \
    apt-get install -y sudo git curl wget make procps python3-pip unzip pandoc jq black

# Install Azure CLI
RUN curl -sL https://aka.ms/InstallAzureCLIDeb | bash

# Python things: update pip, install az-cli
RUN python3 -m pip install pip --upgrade

```

ADD requirements.txt in .devcontainer

``` ini
Authlib==0.15.4
msal==1.12.0
azure-identity==1.6.0
azure-keyvault==4.1.0
azure-keyvault-certificates==4.2.1
fastapi==0.68.1
aiohttp==3.7.4.post0
async-asgi-testclient==1.4.6
python-dotenv==0.17.1
Flask==2.0.1
flask-session==0.3.2
uvicorn==0.14.0
pytest
pytest-asyncio

```

``` dockerfile
# Install required librairies
COPY ./requirements.txt /home/$USERNAME/
RUN pip install -r /home/$USERNAME/requirements.txt
```

TESTS

``` json
		"files.autoSave": "onWindowChange",
		"editor.fontSize": 12,
		"python.testing.pytestArgs": [
			"tests"
		],
		"python.testing.unittestEnabled": false,
		"python.testing.pytestEnabled": true,
		"python.formatting.provider": "black",
		"editor.formatOnSave": true,
```

SECRET

Add azure.sh

``` sh
#!/bin/bash
if [ "$CLIENT_SECRET" != "" ]; then
    exit
fi

if [ "$VAULT_NAME" == "" ] && [ "$VAULT_SECRET_KEY" = "" ]; then
    exit
fi

islogged=$(az account show -o json | jq ".")

if [ "$islogged" == "" ]; then 
    az_login=$(az login -t $TENANT_ID)
    az_account=$(az account set -s $SUBSCRIPTION_ID)
fi

printf "%b\n" "## Checking KeyVault \e[32m$VAULT_NAME\e[0m."
kv=$(az keyvault show -n $VAULT_NAME -o json | jq '.')

if [ "$kv" == "" ]; then
    printf "%b\n" "   KeyVault \e[32m$VAULT_NAME\e[0m does not seems to be reachable."
    exit 3
fi

printf "%b\n" "## Getting vault url"
kv_url=$kv | jq -r '.properties.vaultUri'

printf "%b\n" "## Getting vault secret"
kv_secret=$(az keyvault secret show --vault-name "$VAULT_NAME" --name "$VAULT_SECRET_KEY" --query 'value' --output json | jq -r '.')
if [ "$kv_secret" == "" ]; then
    printf "%b\n" "   Secret \e[32m$VAULT_SECRET_KEY\e[0m does not seems to be reachable."
    exit 3
fi

if ! grep -R "^[#]*\s*CLIENT_SECRET=.*" ~/.bashrc > /dev/null; then
  printf "%b\n" "## Appending \e[32mCLIENT_SECRET\e[0m value in environment variables."
  echo "export CLIENT_SECRET=$kv_secret" >> ~/.bashrc
else
  printf "%b\n" "## Setting \e[32mCLIENT_SECRET\e[0m value because it already exists."
  sed "s,CLIENT_SECRET=[^;]*,export CLIENT_SECRET=$kv_secret," -i ~/.bashrc
fi


```
Uncomment VAULT_NAME and VAULT_SECRET_KEY

``` json
"postStartCommand": "bash .devcontainer/azure.sh"
```