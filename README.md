# **Technical Guide**

## **Deploying Temporal Workers on Azure Kubernetes Service (AKS)**

### **Overview**

Running Temporal Workers in Azure Kubernetes Service (AKS) delivers scalable, resilient, and efficiently managed distributed services. AKS stands out as a leading platform for hosting Temporal workers, offering tight integration with Azure's ecosystem and built-in auto-scaling capabilities with fault tolerance—critical features for enterprise Temporal deployments.

This guide covers the process of deploying and operating Temporal Workers on AKS. The instructions apply to both Temporal self-hosted installations and Temporal Cloud environments.

### **Prerequisites**

Before deploying Workers to AKS, ensure you have:

* Microsoft Azure subscription  
* Provisioned AKS cluster in your Azure subscription  
* Azure CLI installed and configured  
* Docker runtime environment  
* kubectl CLI tool configured for your AKS cluster

### **Project Structure**

This project includes a complete automation system for deploying Temporal workers to AKS:

```
temporal-on-aks-starter/
├── activities.py              # Temporal activities implementation
├── workflows.py               # Temporal workflows definition
├── worker.py                  # Temporal worker implementation
├── client.py                  # Temporal client for triggering workflows
├── config.py                  # Configuration management module
├── crypto_converter.py        # Custom payload converter for encryption
├── keyvault.py                # Azure Key Vault integration for encryption keys
├── config.env                 # Centralized configuration file
├── config.env.example         # Example configuration template
├── source_config.sh           # Script to load configuration
├── generate-k8s-manifests.sh  # Generate K8s manifests from config
├── deploy.sh                  # Complete deployment automation
├── start.sh                   # Local development/k8s startup
├── Dockerfile                 # Container configuration
├── deployment.yaml            # Generated K8s deployment
├── config-map.yaml            # Generated K8s config map
├── acr-secret.yaml            # Generated ACR authentication secret
├── azure-secret.yaml          # Generated Azure secret
├── temporal-secret.yaml       # Generated Temporal secret
├── .gitignore                 # Git ignore rules
└── CONFIGURATION.md           # Detailed configuration documentation
```

### **Getting Started**

This walkthrough demonstrates writing Temporal Worker code, containerizing and publishing the Worker to Azure Container Registry (ACR), then deploying to AKS. Our example leverages Temporal's Python SDK with a complete automation system.

### **Azure Service Principal Setup (Required for Key Vault Access)**

If you're using Azure Key Vault for encryption (which is enabled by default), you need to set up a service principal for authentication:

#### **1. Create a Service Principal**
```bash
az ad sp create-for-rbac --name <your-app-name> --role Contributor --scopes /subscriptions/<subscription-id>
```

This command outputs a JSON object containing:
- `appId` → Use as `AZURE_CLIENT_ID`
- `tenant` → Use as `AZURE_TENANT_ID`  
- `password` → Use as `AZURE_CLIENT_SECRET`

#### **2. Grant Key Vault Access**
For RBAC-enabled Key Vaults (recommended):
```bash
az role assignment create \
  --assignee <service-principal-client-id> \
  --role "Key Vault Secrets User" \
  --scope "/subscriptions/<subscription-id>/resourcegroups/<resource-group>/providers/microsoft.keyvault/vaults/<keyvault-name>"
```

For Access Policy Key Vaults (legacy):
```bash
az keyvault set-policy --name <keyvault-name> --spn <service-principal-client-id> --secret-permissions get list
```

### **Configuration Management**

This project uses a centralized configuration system. All environment variables are managed in `config.env`:

#### **Key Configuration Variables**
- `AZURE_SUBSCRIPTION_ID`: Your Azure subscription ID
- `ACR_NAME`: Azure Container Registry name
- `RESOURCE_GROUP`: Azure resource group name
- `ACR_USERNAME`: ACR username for authentication
- `ACR_PASSWORD`: ACR password for authentication
- `ACR_EMAIL`: Email for ACR authentication
- `TEMPORAL_ADDRESS`: Temporal server address (default: localhost:7233) (for KIND k8s clusters on localhost, set to host.docker.internal:7233)
- `TEMPORAL_NAMESPACE`: Temporal namespace (default: default)
- `TEMPORAL_TASK_QUEUE`: Task queue name (default: test-task-queue)
- `TEMPORAL_API_KEY`: API key for Temporal Cloud authentication
- `KUBERNETES_NAMESPACE`: Kubernetes namespace for deployment
- `APP_NAME`: Application name for deployment
- `APP_IMAGE`: Full image name including ACR path
- `CPU_LIMIT/MEMORY_LIMIT`: Resource limits
- `CPU_REQUEST/MEMORY_REQUEST`: Resource requests
- `AZURE_CLIENT_ID`: Azure service principal client ID (required for Key Vault access)
- `AZURE_TENANT_ID`: Azure tenant ID (required for Key Vault access)
- `AZURE_CLIENT_SECRET`: Azure service principal client secret (required for Key Vault access)
- `KEYVAULT_URL`: Azure Key Vault URL (required for encryption)
- `KEYVAULT_SECRET_NAME`: Name of the secret in Key Vault containing the encryption key

#### **Setup Configuration**
```bash
# Copy the example configuration
cp config.env.example config.env

# Edit with your actual values
nano config.env
```

### **Writing Worker Implementation**

Temporal applications separate business logic into Workflow definitions while Worker processes handle execution of Workflows and Activities.

#### **Activities (`activities.py`)**
```python
from temporalio import activity

@activity.defn
async def your_first_activity(name: str) -> str:
    return f"Hello, {name}!"

@activity.defn
async def your_second_activity(data: str) -> str:
    return f"Processed: {data}"

@activity.defn
async def your_third_activity(result: str) -> str:
    return f"Final result: {result}"
```

#### **Workflows (`workflows.py`)**
```python
from datetime import timedelta
from temporalio import workflow

@workflow.defn
class YourWorkflow:
    @workflow.run
    async def run(self, name: str) -> str:
        return await workflow.execute_activity(
            your_first_activity,
            name,
            start_to_close_timeout=timedelta(seconds=10),
        )

your_workflow = YourWorkflow
```

#### **Worker (`worker.py`)**
The worker uses the centralized configuration system:

```python
import asyncio
import os

from temporalio.worker import Worker
from temporalio.client import Client

from workflows import your_workflow
from activities import your_first_activity, your_second_activity, your_third_activity
from config import TEMPORAL_ADDRESS, TEMPORAL_NAMESPACE, TEMPORAL_TASK_QUEUE, TEMPORAL_API_KEY, KEYVAULT_URL, KEYVAULT_SECRET_NAME
from crypto_converter import encrypted_converter

async def main():
    # For local development, connect without TLS or API key
    if TEMPORAL_ADDRESS.startswith("localhost") or "host.docker.internal" in TEMPORAL_ADDRESS:
        client = await Client.connect(
            TEMPORAL_ADDRESS,
            namespace=TEMPORAL_NAMESPACE,
            data_converter=encrypted_converter
        )
    else:
        # For Temporal Cloud, use TLS and API key
        client = await Client.connect(
            TEMPORAL_ADDRESS,
            namespace=TEMPORAL_NAMESPACE,
            rpc_metadata={"temporal-namespace": TEMPORAL_NAMESPACE},
            api_key=TEMPORAL_API_KEY,
            tls=True,
            data_converter=encrypted_converter
        )
    
    print("Initializing worker...")
    worker = Worker(
        client,
        task_queue=TEMPORAL_TASK_QUEUE,
        workflows=[your_workflow],
        activities=[
            your_first_activity,
            your_second_activity,
            your_third_activity
        ]
    )

    print("Starting worker... Awaiting tasks.")
    await worker.run()

if __name__ == "__main__":
    asyncio.run(main())
```

#### **Client (`client.py`)**
A client application that triggers workflows:

```python
import asyncio
import datetime
import logging
from temporalio.client import Client, WorkflowHandle

from workflows import your_workflow
from config import TEMPORAL_ADDRESS, TEMPORAL_NAMESPACE, TEMPORAL_TASK_QUEUE, TEMPORAL_API_KEY, KEYVAULT_URL, KEYVAULT_SECRET_NAME
from crypto_converter import encrypted_converter

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

async def main():
    try:
        logging.info("Connecting to Temporal server...")
        
        # For local development, connect without TLS or API key
        if TEMPORAL_ADDRESS.startswith("localhost") or "host.docker.internal" in TEMPORAL_ADDRESS:
            client = await Client.connect(
                TEMPORAL_ADDRESS,
                namespace=TEMPORAL_NAMESPACE,
                data_converter=encrypted_converter
            )
        else:
            # For Temporal Cloud, use TLS and API key
            client = await Client.connect(
                TEMPORAL_ADDRESS,
                namespace=TEMPORAL_NAMESPACE,
                rpc_metadata={"temporal-namespace": TEMPORAL_NAMESPACE},
                api_key=TEMPORAL_API_KEY,
                tls=True,
                data_converter=encrypted_converter
            )
            
        logging.info(f"Successfully connected to Temporal server at {TEMPORAL_ADDRESS} in namespace {TEMPORAL_NAMESPACE}.")
    except Exception as e:
        logging.error(f"Failed to connect to Temporal server: {e}")
        return

    workflow_counter = 0
    while True:
        workflow_counter += 1
        workflow_id = f"your-workflow-{workflow_counter}-{datetime.datetime.now().strftime('%Y%m%d%H%M%S')}"
        workflow_arg = f"ClientTriggeredPayload-{workflow_counter}"

        try:
            logging.info(f"Starting workflow with ID: {workflow_id} and argument: {workflow_arg}...")
            result_handle: WorkflowHandle[str] = await client.start_workflow(
                your_workflow.run,
                workflow_arg,
                id=workflow_id,
                task_queue=TEMPORAL_TASK_QUEUE,
            )
            logging.info(f"Workflow {workflow_id} started. Run ID: {result_handle.result_run_id}")

        except Exception as e:
            logging.error(f"Failed to start workflow {workflow_id}: {e}")

        logging.info("Waiting 10 seconds before next workflow execution...")
        await asyncio.sleep(10)

if __name__ == "__main__":
    asyncio.run(main())
```

### **Container Preparation for Kubernetes**

Containerization is required for Kubernetes deployment. The project includes a Dockerfile:

```dockerfile
# Use Python 3.11 slim image as base
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Install Temporal Python SDK
RUN pip install --no-cache-dir temporalio requests

# Copy application code
COPY *.py /app/
COPY start.sh /app/start.sh

# Configure Python for unbuffered output
ENV PYTHONUNBUFFERED=1

# Ensure the script is executable
RUN chmod +x /app/start.sh

# Use the script to start both processes
CMD ["bash", "/app/start.sh"]
```

### **Automated Deployment Process**

This project includes a complete automation system for deploying to AKS:

#### **1. Configuration Setup**
```bash
# Copy and configure the environment file
cp config.env.example config.env
# Edit config.env with your actual values
```

#### **2. Deploy to AKS**
```bash
# Make scripts executable
chmod +x deploy.sh generate-k8s-manifests.sh source_config.sh

# Run the complete deployment
./deploy.sh
```

The `deploy.sh` script automates:
- Building Docker image for multiple architectures (linux/amd64, linux/arm64)
- Logging into Azure Container Registry
- Tagging and pushing the image to ACR
- Creating Kubernetes namespace
- Generating Kubernetes manifests from configuration
- Applying secrets (ACR, Azure, and Temporal) and config maps
- Deploying the application to AKS

#### **3. OPTIONAL: Manual Deployment Steps (if so inclined, if not, skip to Local Development topic)**

If you prefer manual deployment, the automation scripts can be run individually:

**Generate Kubernetes Manifests:**
```bash
./generate-k8s-manifests.sh
```

This script generates:
- `acr-secret.yaml` - ACR authentication secret
- `azure-secret.yaml` - Azure secret
- `temporal-secret.yaml` - Temporal secret
- `config-map.yaml` - Temporal configuration
- `deployment.yaml` - Application deployment

**Apply Manifests:**
```bash
# Source the configuration
source ./source_config.sh

# Create namespace
kubectl create namespace $KUBERNETES_NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

# Apply secrets and config
kubectl apply -f acr-secret.yaml --namespace $KUBERNETES_NAMESPACE
kubectl apply -f azure-secret.yaml --namespace $KUBERNETES_NAMESPACE
kubectl apply -f temporal-secret.yaml --namespace $KUBERNETES_NAMESPACE
kubectl apply -f config-map.yaml --namespace $KUBERNETES_NAMESPACE

# Deploy the application
kubectl apply -f deployment.yaml --namespace $KUBERNETES_NAMESPACE
```

### **Local Development**

For local development and testing:

#### **Prerequisites**
- Python 3.11+
- Temporal server running locally or Temporal Cloud access

#### **Setup**
```bash
# Install dependencies
pip install temporalio

# Copy and configure the environment file
cp config.env.example config.env
# Edit config.env for local development

# IMPORTANT: Always source your config before running Python!
# This ensures KEYVAULT_URL, KEYVAULT_SECRET_NAME, and all other variables are set.
source ./source_config.sh

# Run the worker
python worker.py &

# In another terminal (with config sourced), run the client
python client.py
```

#### **Quick Start Script**
```bash
# Use the provided start script (this will source all config for you)
./start.sh
```

> **Note:** If you run `python worker.py` or `python client.py` directly, without first sourcing `./source_config.sh` or using `./start.sh`, required environment variables (like `KEYVAULT_URL`) will NOT be set and you will get a KeyError. Always use `./start.sh` or manually source the config before running Python scripts.

### **Validating Worker Connectivity**

Post-deployment, verify Workers have established connection to Temporal:

```bash
# Check pod status
kubectl get pods -n $KUBERNETES_NAMESPACE

# Examine Worker logs
kubectl logs <pod-name> -n $KUBERNETES_NAMESPACE
```

Successful connection displays:
```
Initializing worker...
Starting worker... Awaiting tasks.
```

### **Resource Requirements**

The Kubernetes deployment is configured with:
- **Requests**: 0.2 CPU, 256Mi memory
- **Limits**: 0.5 CPU, 512Mi memory

These values can be adjusted in `config.env` under the Resource Limits section.

### **Configuration Management Details**

For complete configuration details, see [CONFIGURATION.md](CONFIGURATION.md).

The configuration system supports:
- Environment variable priority (system env > config.env > defaults)
- Validation of required variables
- Automatic manifest generation
- Support for both local development and production deployments

### **Troubleshooting**

#### **Common Issues**

1. **Configuration Errors**: Ensure all required variables are set in `config.env`
2. **ACR Authentication**: Verify ACR credentials in the configuration
3. **Temporal Connection**: Check Temporal server address and API key
4. **Resource Limits**: Adjust CPU/memory limits if pods fail to start

#### **Useful Commands**

```bash
# Check configuration
source ./source_config.sh

# View generated manifests
cat deployment.yaml
cat config-map.yaml

# Check pod logs
kubectl logs -f deployment/$APP_NAME -n $KUBERNETES_NAMESPACE

# Restart deployment
kubectl rollout restart deployment/$APP_NAME -n $KUBERNETES_NAMESPACE
```

Your Temporal Worker is now operational on AKS with a complete automation system for deployment and configuration management. The project provides a production-ready foundation for running Temporal workflows in Kubernetes with proper resource management, configuration handling, and deployment automation.

## Optional: End-to-End Payload Encryption with Azure Key Vault

This section describes how to transparently encrypt all Temporal workflow/activity payloads using a symmetric key stored in Azure Key Vault. This is **optional**—the rest of the project works without it.

### Overview

- All workflow and activity payloads are encrypted at rest and in transit using AES-GCM.
- The encryption key is securely fetched at runtime from Azure Key Vault.
- No key material is stored in code or config files.

### Additional Setup

#### 1. Install Additional Dependencies

```bash
pip install azure-identity azure-keyvault-secrets cryptography
```

#### 2. Add Key Vault Configuration

Add these to your `config.env` (see `config.env.example`