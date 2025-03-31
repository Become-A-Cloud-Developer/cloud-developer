+++
title = "1. Run .Net Webapp in a Docker Container"
weight = 1
date = 2025-02-10
draft = false
+++

# Run .Net Webapp in a Docker Container

## Overview

In this exercise, you will build a docker image and deploy the **CloudSoft** web application to a VM configured as a _Docker Host_. We will use Docker Hub as registry.

## Prerequisites

- **Azure Account** with an active subscription.
- **Docker Hub** The official Docker registry.
- **Docker Installed** for local container builds and testing.
- **Azure CLI Installed** to interact with Azure resources.

## Step-by-step Guide

### Step 1: Create a Dockerfile

This step is only necessary if you don´t have a Dockerfile already

1. Run the following command to create a new file named `Dockerfile`. Follow the wizard in the terminal:

```bash
docker init
```

Press `<Enter>` after each question.

```
> ASP.NET Core - (detected) suitable for an ASP.NET Core application
> CloudSoft
? What version of .NET do you want to use? (9.0)
? What port do you want to use to access your app? (8080)
```

### Step 2: Login to Docker and Docker Hub

1. Go to Docker Hub and sign up for an account if you don´t have one already
2. Create an Access Token
	
	- Go to **Account settings** -> **Personal access tokens**
	- Generate new token
	- Access permission: Read & Write
	- Click **Generate**
	- Open a terminal and follow the login instructions

### Step 3: Build and Push the Container Image (Windows & Mac with Intel)

1. Navigate to your project directory and build the Docker image:

   ```bash
   docker build -t <username>/cloudsoft:latest .
   ```
   
2. Push the image to ACR:

   ```sh
   docker push <username>/cloudsoft:latest
   ```

### Step 3: Build and Push the Container Image (Mac with Apple Silicon)

1. Change the Dockerfile. Replace the dotnet publish command to this:

	```yaml
	dotnet publish --runtime linux-${TARGETARCH} --self-contained false -o /app
	```

	> This is what is replaced:
	> 
	> ```yaml
	> dotnet publish -a ${TARGETARCH/amd64/x64} --use-current-runtime --self-contained false -o /app
	> ```

2. Create a multi-architecture image builder:

	```bash
	docker buildx create --use
	```

3. Navigate to your project directory. Build and push the Docker image:

   ```bash
   docker buildx build --platform linux/amd64,linux/arm64 -t <username>/cloudsoft:latest --push .
   ```
	> Since multi-arch images cannot be stored locally, you must push them directly to a container registry (ACR, Docker Hub, etc.).


### Step 4: Provision a VM as Docker Host

1. **Create a Cloud-Init Configuration**:

    Save the following configuration in a file named `cloud-init_docker.yaml`. This script will install Docker on your Ubuntu VM.

	> `cloud-init_docker.yaml`

	```yaml
	#cloud-config

	# Install Docker Engine
	runcmd:
	# Add Docker's official GPG key
	- apt-get update -y
	- apt-get install ca-certificates curl -y
	- install -m 0755 -d /etc/apt/keyrings
	- curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
	- chmod a+r /etc/apt/keyrings/docker.asc

	# Add the repository to Apt sources
	- echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

	# Install Docker
	- apt-get update -y
	- apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
	```

2. **Provision Your VM with Azure CLI**:

    Execute the following script to create your VM and apply your Docker setup:

	> `provision_docker_vm.sh`

	```bash
	#!/bin/bash
	
	set -e  # Exit immediately if a command exits with a non-zero status.
	
	# Variables - adjust these as needed
	RESOURCE_GROUP="CloudSoftRG"
	LOCATION="northeurope"          # Change to your preferred Azure region
	
	VM_NAME="CloudSoftVM"
	ADMIN_USERNAME="azureuser"
	SSH_KEY_PATH="$HOME/.ssh/id_rsa.pub"  # Path to your SSH public key
	PORT=80
	
	# Function to check prerequisites
	check_prerequisites() {
	  # Check if the SSH key exists
	  if [ ! -f "$SSH_KEY_PATH" ]; then
	    echo "SSH key not found at $SSH_KEY_PATH. Please generate an SSH key (e.g., with 'ssh-keygen')."
	    exit 1
	  fi
	  echo "✔ SSH key found at $SSH_KEY_PATH"
	
	  # Check if the Azure CLI is installed
	  if ! command -v az &> /dev/null; then
	    echo "Azure CLI is not installed. Please install it first."
	    exit 1
	  fi
	  echo "✔ Azure CLI is installed."
	
	  # Check if the Azure CLI is logged in
	  if [ -z "$(az account show --query id -o tsv)" ]; then
	    echo "Azure CLI is not logged in. Please run 'az login' first."
	    exit 1
	  fi
	  echo "✔ Azure CLI is logged in."
	}
	
	# Run the prerequisite checks
	check_prerequisites
	
	# Create a resource group
	echo "Creating resource group '$RESOURCE_GROUP' in region '$LOCATION'..."
	az group create \
	    --name "$RESOURCE_GROUP" \
	    --location "$LOCATION"
	
	# Create the Ubuntu VM
	echo "Creating Ubuntu VM '$VM_NAME'..."
	az vm create \
	  --resource-group "$RESOURCE_GROUP" \
	  --location northeurope \
	  --name "$VM_NAME" \
	  --image Ubuntu2204 \
	  --size Standard_B1s \
	  --admin-username "$ADMIN_USERNAME" \
	  --generate-ssh-keys \
	  --custom-data @cloud-init_docker.yaml
	
	# Open port on the VM to allow incoming traffic
	echo "Opening port '$PORT' on the VM '$VM_NAME' ..."
	az vm open-port \
	  --resource-group "$RESOURCE_GROUP" \
	  --name "$VM_NAME" \
	  --port $PORT \
	  --priority 1001
	
	echo "Setup complete. Your VM '$VM_NAME' is ready and port '$PORT' is open."
	echo "Connect to your VM with: ssh $ADMIN_USERNAME@$(az vm show -d -g $RESOURCE_GROUP -n $VM_NAME --query publicIps -o tsv)"
	
	```

    > Make sure to replace `<resource_group>`, `<vm_name>`, and `<vm_port>` with your specific values.

	Finally run the provisioning script:

	```bash
	./provision_docker_vm.sh
	```

	> Note: Don't forget to `chmod +x provision_docker_vm.sh ` if you are on Linux or Mac


3. **Verify Installation**:

	> Note: It can take a few minutes to install Docker, so be patient

    SSH into your VM and run:

    ```bash
    sudo docker run hello-world
    ```

    This command verifies Docker is installed and running correctly.
    
4. **Run the containerized CloudSoft webapp**
	
	```bash
	sudo docker run -d -p 80:8080 <username>/cloudsoft
	```
	
3. **Verify**:

    Open a browser and go to the public IP of the VM
    
    You should see your application running.

# docker push! 🚀

