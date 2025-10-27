+++
title = 'Creating Self-Hosted Azure DevOps Agents with Azure Container App Jobs and Managed Identity'
date = '2025-02-17T09:00:00+00:00'
draft = false
slug = 'azure-devops-agents-container-app-jobs'
tags = ['Azure DevOps', 'Containers', 'Security', 'Microsoft', 'Azure']
description = 'With system-assigned managed identities, Azure Container App Jobs can securely access Azure DevOps without the need for a PAT token'
+++

Using container app jobs for self-hosted Azure DevOps agents allows for more control over what is running on your DevOps agents. Both VMSS and the newer managed DevOps pools give you the option to run agents on your own virtual network which is excellent for securing network traffic but if you also need to have control what is running on them then configuring the agents with docker in a container app job is a good option. You also have the added security of Defender for containers integration to ensure you can keep your images secure.

One of the downsides to container apps/jobs as agents until recently has been the requirement to use a PAT token to authenticate with an Azure DevOps agent pool. PAT is not ideal as the token can be exposed, can expire and is just all round vulnerable to cyberattacks. If an attacker gets their hands on your PAT token then your at risk of unauthorized access to your source code, pipelines and and ultimately your cloud infrastructure resources.

Microsoft have also recently annouced that they are distancing themselves from PAT in Azure DevOps: [Reducing personal access token (PAT) usage across Azure DevOps](https://devblogs.microsoft.com/devops/reducing-pat-usage-across-azure-devops/)

We can now remove this risk by configuring system-assigned managed identity on a container app job to run as an Azure DevOps agent.

So lets get started:

### Pre-requisites

- Install [Windows Subsystem for Linux](https://learn.microsoft.com/en-us/windows/wsl/install) (WSL) and install Docker desktop or the Docker engine directly in WSL to be able to build your image.
- Deploy a virtual network and subnet in Azure for the container app environment (if using your own vnet).
- Deploy an Azure container registry.

## Dockerfile

First we need to build our dockerfile to use as an Azure DevOps agent. For this I recommend using the dockerfile which uses the 'python:3-alpine' OS that is documented by Microsoft in this article: [Run a self-hosted agent in Docker - Azure Pipelines](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops#create-and-build-the-dockerfile-1)

```dockerfile
FROM python:3-alpine
ENV TARGETARCH="linux-musl-x64"

# Another option:
# FROM arm64v8/alpine
# ENV TARGETARCH="linux-musl-arm64"

RUN apk update && \
  apk upgrade && \
  apk add bash curl gcc git icu-libs jq musl-dev python3-dev libffi-dev openssl-dev cargo make

# Install Azure CLI
RUN pip install --upgrade pip
RUN pip install azure-cli

WORKDIR /azp/

COPY ./start.sh ./
RUN chmod +x ./start.sh

RUN adduser -D agent
RUN chown agent ./
USER agent

# Another option is to run the agent as root.
# ENV AGENT_ALLOW_RUNASROOT="true"

ENTRYPOINT [ "./start.sh" ]
```

Next we need to configure the start.sh file. The start.sh file in the Microsoft documentation uses a service principal to configure the $AZP_TOKEN value but this requires that you store a client secret somewhere. Since Azure DevOps supports adding managed identities as users then I have configured the file as follows:

- Configure $AZP_TOKEN to use managed identity instead of using a service principal or PAT token.
- Configure $AZP_PLACEHOLDER cleanup to keep the placeholder configuration if '$AZP_PLACEHOLDER = 1' is present which is required for KEDA scaling.

Note: for those of you wondering, the 'APPLICATION_ID' is the ID refers to the Azure DevOps app registration from Microsofts own tenant.

```bash
#!/bin/bash
set -e

if [ -z "$AZP_URL" ]; then
  echo 1>&2 "error: missing AZP_URL environment variable"
  exit 1
fi

IDENTITY_HEADER="$IDENTITY_HEADER"
IDENTITY_ENDPOINT="$IDENTITY_ENDPOINT"
APPLICATION_ID="499b84ac-1321-427f-aa17-267ca6975798"

response=$(curl -s -X GET -H "X-IDENTITY-HEADER: $IDENTITY_HEADER" "$IDENTITY_ENDPOINT?resource=$APPLICATION_ID&api-version=2019-08-01")
AZP_TOKEN=$(echo "$response" | jq -r '.access_token')

if [ -z "$AZP_TOKEN_FILE" ]; then
  if [ -z "$AZP_TOKEN" ]; then
    echo 1>&2 "error: missing AZP_TOKEN environment variable"
    exit 1
  fi
  AZP_TOKEN_FILE=/azp/.token
  echo -n $AZP_TOKEN > "$AZP_TOKEN_FILE"
fi

unset AZP_TOKEN

if [ -n "$AZP_WORK" ]; then
  mkdir -p "$AZP_WORK"
fi

export AGENT_ALLOW_RUNASROOT="1"

cleanup() {
  if [ -e config.sh ]; then
    if [ -z "$AZP_PLACEHOLDER" ]; then
      print_header "Cleanup. Removing Azure Pipelines agent..."
      
      # If the agent has some running jobs, the configuration removal process will fail.
      # So, give it some time to finish the job.
      while true; do
        ./config.sh remove --unattended --auth PAT --token $(cat "$AZP_TOKEN_FILE") && break
        echo "Retrying in 30 seconds..."
        sleep 30
      done
    else
      print_header "Cleanup skipped as Agent is marked as a placeholder... this option should be removed if Azure DevOps allows queueing to empty Agent Pools."
    fi
  fi
}

print_header() {
  lightcyan='\033[1;36m'
  nocolor='\033[0m'
  echo -e "${lightcyan}$1${nocolor}"
}

# Let the agent ignore the token env variables
export VSO_AGENT_IGNORE=AZP_TOKEN,AZP_TOKEN_FILE

print_header "1. Determining matching Azure Pipelines agent..."

AZP_AGENT_PACKAGES=$(curl -LsS \
    -u user:$(cat "$AZP_TOKEN_FILE") \
    -H 'Accept:application/json;' \
    "$AZP_URL/_apis/distributedtask/packages/agent?platform=$TARGETARCH&top=1")

AZP_AGENT_PACKAGE_LATEST_URL=$(echo "$AZP_AGENT_PACKAGES" | jq -r '.value[0].downloadUrl')

if [ -z "$AZP_AGENT_PACKAGE_LATEST_URL" -o "$AZP_AGENT_PACKAGE_LATEST_URL" == "null" ]; then
  echo 1>&2 "error: could not determine a matching Azure Pipelines agent"
  echo 1>&2 "check that account '$AZP_URL' is correct and the token is valid for that account"
  exit 1
fi

print_header "2. Downloading and extracting Azure Pipelines agent..."

curl -LsS $AZP_AGENT_PACKAGE_LATEST_URL | tar -xz & wait $!

source ./env.sh

trap 'cleanup; exit 0' EXIT
trap 'cleanup; exit 130' INT
trap 'cleanup; exit 143' TERM

print_header "3. Configuring Azure Pipelines agent..."

./config.sh --unattended \
  --agent "${AZP_AGENT_NAME:-$(hostname)}" \
  --url "$AZP_URL" \
  --auth PAT \
  --token $(cat "$AZP_TOKEN_FILE") \
  --pool "${AZP_POOL:-Default}" \
  --work "${AZP_WORK:-_work}" \
  --replace \
  --acceptTeeEula & wait $!

print_header "4. Running Azure Pipelines agent..."

./run.sh "$@" & wait $!
```

Now you can run docker build and push commands to your container registry directly from WSL or via an Azure DevOps pipeline task with a service connection that has the 'acrPush' role on the container registry.

```bash
# build docker file with tag
docker build --tag "agent:latest" --file "./dockerfile" .

# tag docker file for use in acr
docker tag agent:latest <name_of_container_registry>.azurecr.io/agent:latest

# push docker file to acr
docker push <name_of_container_registry>.azurecr.io/agent:latest
```

## Azure container app job

Now we are ready to deploy the container app job thats going to run the Azure DevOps agent.

For this I am using bicep code to deploy both a container app environment and a container app job. There are a few different options for doing this deployment, you can either deploy the app environment on a Microsoft hosted network or you can deploy it in an isolated subnet with either a consumption workload or a dedicated workload profile (or both).

In the following example I'm deploying on my own virtual network in a dedicated subnet that is using an internal load balancer on the container environment and a D4 workload consumption profile. Here are some points also worth mentioning in the configuration:

- Scale rules are set to use [KEDA scaling](https://keda.sh/docs/2.16/scalers/azure-pipelines/) with the managed identity of the container app job. This requires that the trigger parameter is the AzureDevOps URL of your organization and that it is stored as a secret in the container app config.
- The managed identity needs to be added as a user to Azure DevOps wiht stakeholder access and added to the DevOps Agent pool as an administrator.
- You need to give the managed identity of the container app job 'acrPull' access on the container registry you are using to pull the image from. With this you can avoid using the admin user access key on the container registry.
- You also need to deploy a placeholder agent (as mentioned earlier) which will be present in the Azure DevOps agent pool as an offline resource. This is required so that when you trigger a pipeline the KEDA scaling will start the app job. This can be deleted once its been registrered offline but you will need to add the managed identity for that to the pool as well to ensure it works before you delete the app.

## Summary

Using this configuration will ensure that you can remove the need for PAT on your Azure DevOps agents and take advantage of the system-assigned managed identities on the container app jobs instead.
