+++
title = "1. Implementing Azure Key Vault with MongoDB on Azure VM"
weight = 1
date = 2025-03-24
draft = false
+++

# Exercise 1: Implementing Azure Key Vault with MongoDB on Azure VM

## Goal

Implement Azure Key Vault for secure secret management in your ASP.NET Core application, connecting to MongoDB/Cosmos DB and deploying to an Azure Ubuntu VM with managed identity for secure authentication.

## Learning Objectives

By the end of this exercise, you will:

- Provision **Azure Key Vault** and **Azure Cosmos DB for MongoDB API**
- Configure **secret management** for sensitive connection strings
- Implement the **Options Pattern** for configuration
- Use **feature flags** to enable different services based on environment
- Setup **managed identity** for VM-to-Key Vault authentication
- Deploy an ASP.NET Core application to an **Azure Ubuntu VM**
- Understand **configuration provider precedence** in ASP.NET Core

## Prerequisites

- An Azure account with active subscription
- Azure CLI installed
- .NET 9.0 SDK installed
- Docker and Docker Compose installed
- Basic understanding of ASP.NET Core MVC applications
- Completed Exercise 5 or have a similar application structure

## Understanding Azure Key Vault in ASP.NET Core Applications

### Why Use Azure Key Vault?

1. **Secure Secret Management**: Store sensitive configuration data like connection strings and API keys securely
2. **Centralized Configuration**: Manage secrets across multiple applications and environments
3. **Access Control**: Fine-grained control over who or what can access specific secrets
4. **Audit Trail**: Track when and by whom secrets are accessed
5. **Rotation**: Easily rotate credentials without redeploying applications

### How Azure Key Vault Integrates with ASP.NET Core

ASP.NET Core uses a configuration system based on key-value pairs from various providers. The configuration providers are applied in a specific order, with later providers overriding values from earlier ones. Azure Key Vault is typically one of the last providers registered, allowing it to override settings from appsettings.json files.

### Configuration Provider Precedence

In ASP.NET Core, configuration providers are applied in the following order (later providers override earlier ones):

1. Default values (hardcoded in the application)
2. Command-line arguments
3. Environment variables (prefixed with `ASPNETCORE_`)
4. User secrets (in development)
5. appsettings.json
6. appsettings.{Environment}.json (e.g., appsettings.Development.json)
7. **Azure Key Vault** (when configured)

This precedence means that values in Azure Key Vault will override any conflicting values from appsettings files, providing a secure way to manage environment-specific configuration.

## Step-by-Step Instructions

### Phase I: Setting Up Azure Resources and Local Configuration

### Step 1: Provision Azure Cosmos DB with MongoDB API

1. Log in to the Azure Portal and create a new Azure Cosmos DB account:

    ```bash
    #!/bin/bash

    resource_group=CloudSoftRG
    db_name=cloudsoft-mongodb # Must be globally unique

    # Create resource group
    az group create --location northeurope --name $resource_group

    # Create Cosmos DB account with MongoDB API
    az cosmosdb create \
    --name $db_name \
    --resource-group $resource_group \
    --kind MongoDB \
    --capabilities EnableServerless \
    --default-consistency-level Session \
    --server-version 7.0
    ```

2. After creation, navigate to the Azure Portal, go to your Cosmos DB account, and note the following:
   - Under "Connection strings", copy the Primary Connection String
   - This will be used later as a secret in Azure Key Vault

    ```bash
    az cosmosdb keys list \
        --name cloudsoft-mongodb \
        --resource-group CloudSoftRG \
        --type connection-strings \
        --query "connectionStrings[?description=='Primary MongoDB Connection String'].connectionString" \
        --output tsv
    ```

### Step 2: Provision Azure Key Vault

1. Create an Azure Key Vault in the same resource group:

   ```bash
    #!/bin/bash
    
    resource_group=CloudSoftRG2
    vault_name=cloudsoftkeyvault$(date +%d%m%y)
    
    # Create resource group
    az group create --location northeurope --name $resource_group
    
    # Create Key Vault
    az keyvault create \
        --name $vault_name \
        --resource-group $resource_group \
        --location northeurope
    
    # Get your user principal ID
    USER_ID=$(az ad signed-in-user show --query id --output tsv)
    
    # Assign Key Vault Administrator role to yourself
    az role assignment create \
      --assignee $USER_ID \
      --role "Key Vault Administrator" \
      --scope $(az keyvault show --name $vault_name --resource-group $resource_group --query id -o tsv)
   ```

   > 💡 **Information**
   >
   > - The date suffix in the Key Vault name ensures uniqueness
   > - Key Vault names must be globally unique across all of Azure

2. Assign yourself the "Key Vault Administrator" role (if you didn't in step 1):

   ```bash
   # Get your user principal ID
   USER_ID=$(az ad signed-in-user show --query id --output tsv)

   # Assign Key Vault Administrator role
   az role assignment create \
     --assignee $USER_ID \
     --role "Key Vault Administrator" \
     --scope $(az keyvault show --name cloudsoftkeyvault$(date +%d%m%y) --resource-group CloudSoftRG --query id -o tsv)
   ```

3. Note the Key Vault URI (you'll need it later):

   ```bash
   az keyvault show \
     --name cloudsoftkeyvault$(date +%d%m%y) \
     --resource-group CloudSoftRG \
     --query properties.vaultUri \
     --output tsv
   ```

### Step 3: Configure Application Settings for Key Vault

1. Update `appsettings.json` to include placeholders for Azure Key Vault:

   > `appsettings.json`

   ```json
   {
     "Logging": {
       "LogLevel": {
         "Default": "Information",
         "Microsoft.AspNetCore": "Warning"
       }
     },
     "AllowedHosts": "*",
     "FeatureFlags": {
       "UseMongoDb": false,
       "UseAzureStorage": false,
       "UseAzureKeyVault": false
     },
     "MongoDb": {
       "ConnectionString": "mongodb://{username}:{password}@{hostname}:{port}",
       "DatabaseName": "cloudsoft",
       "SubscribersCollectionName": "subscribers"
     },
     "AzureKeyVault": {
       "KeyVaultUri": "https://{keyvaultname}.vault.azure.net/"
     }
   }
   ```

2. Update `appsettings.Development.json` with the actual Key Vault URI and enable MongoDB and Key Vault feature flags:

   > `appsettings.Development.json`

   ```json
   {
     "Logging": {
       "LogLevel": {
         "Default": "Information",
         "Microsoft.AspNetCore": "Warning"
       }
     },
     "FeatureFlags": {
       "UseMongoDb": true,
       "UseAzureStorage": false,
       "UseAzureKeyVault": true
     },
     "MongoDb": {
       "ConnectionString": "mongodb://root:example@localhost:27017",
       "DatabaseName": "cloudsoft",
       "SubscribersCollectionName": "subscribers"
     },
     "AzureKeyVault": {
       "KeyVaultUri": "https://cloudsoftkeyvault$(date +%d%m%y).vault.azure.net/"
     }
   }
   ```

3. Create `docker-compose.yml` for local MongoDB instance:

   > `docker-compose.yml`

   ```yaml
   services:
     mongodb:
       image: mongo:latest
       container_name: mongodb
       restart: always
       ports:
         - "27017:27017"
       environment:
         MONGO_INITDB_ROOT_USERNAME: ${MONGO_USERNAME:-root}
         MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD:-example}
       volumes:
         - mongodb_data:/data/db
       networks:
         - mongo_network

     mongo-express:
       image: mongo-express:latest
       container_name: mongo-express
       restart: always
       ports:
         - "8081:8081"
       environment:
         ME_CONFIG_MONGODB_ADMINUSERNAME: ${MONGO_USERNAME:-root}
         ME_CONFIG_MONGODB_ADMINPASSWORD: ${MONGO_PASSWORD:-example}
         ME_CONFIG_MONGODB_SERVER: mongodb
       depends_on:
         - mongodb
       networks:
         - mongo_network

   volumes:
     mongodb_data:

   networks:
     mongo_network:
       driver: bridge
   ```

4. Start the local MongoDB instance:

   ```bash
   docker-compose up -d
   ```

### Step 4: Create Azure Key Vault Options Class

1. Create a `Configurations` directory if it doesn't exist yet:

   ```bash
   mkdir -p Configurations
   ```

2. Create an options class for Azure Key Vault configuration:

   > `Configurations/AzureKeyVaultOptions.cs`

   ```csharp
   namespace CloudSoft.Configurations;

   public class AzureKeyVaultOptions
   {
       public const string SectionName = "AzureKeyVault";
       public string? KeyVaultUri { get; set; }
   }
   ```

   > 💡 **Information**
   >
   > - Using the Options Pattern allows for strongly-typed access to configuration sections
   > - The `SectionName` constant helps maintain consistency when referencing this configuration section

### Step 5: Add Required NuGet Packages

1. Add the necessary packages for Azure Key Vault integration:

   ```bash
   dotnet add package Azure.Extensions.AspNetCore.Configuration.Secrets
   dotnet add package Azure.Identity
   dotnet add package Azure.Security.KeyVault.Secrets
   ```

   > 💡 **Information**
   >
   > - `Azure.Extensions.AspNetCore.Configuration.Secrets` provides the configuration provider
   > - `Azure.Identity` provides authentication mechanisms including DefaultAzureCredential
   > - `Azure.Security.KeyVault.Secrets` is the client library for Key Vault secrets

### Step 6: Register Azure Key Vault Configuration Provider

1. Update `Program.cs` to register Azure Key Vault as a configuration provider:

    > `Program.cs`

    ```csharp
    ...

      using Azure.Identity;
      
      ...

      var builder = WebApplication.CreateBuilder(args);

    ...

      // Check if Azure Key Vault should be used
      bool useAzureKeyVault = builder.Configuration.GetValue<bool>("FeatureFlags:UseAzureKeyVault");

      if (useAzureKeyVault)
      {
          // Configure Azure Key Vault options
          builder.Services.Configure<AzureKeyVaultOptions>(
              builder.Configuration.GetSection(AzureKeyVaultOptions.SectionName));

          // Get Key Vault URI from configuration
          var keyVaultOptions = builder.Configuration
              .GetSection(AzureKeyVaultOptions.SectionName)
              .Get<AzureKeyVaultOptions>();
          var keyVaultUri = keyVaultOptions?.KeyVaultUri;

          // Register Azure Key Vault as configuration provider
          if (string.IsNullOrEmpty(keyVaultUri))
          {
              throw new InvalidOperationException("Key Vault URI is not configured.");
          }

          builder.Configuration.AddAzureKeyVault(
              new Uri(keyVaultUri),
              new DefaultAzureCredential());

          Console.WriteLine("Using Azure Key Vault for configuration");
      }
      
      ...
      
      ```

   > 💡 **Information**
   >
   > - The `DefaultAzureCredential` will handle authentication to Azure Key Vault
   > - In development, it uses your Azure CLI credentials
   > - In production, it will use the VM's managed identity

### Step 7: Add MongoDB Connection String as Secret in Azure Key Vault

1. Add the Cosmos DB connection string as a secret in Key Vault:

   ```bash
   # Replace with your actual connection string from Cosmos DB
   CONNECTION_STRING="mongodb://cloudsoft-mongodb:your-primary-password@cloudsoft-mongodb.mongo.cosmos.azure.com:10255/?ssl=true&replicaSet=globaldb&retrywrites=false&maxIdleTimeMS=120000&appName=@cloudsoft-mongodb@"

   # Add the secret to Key Vault
   # Note: We use -- instead of : for secret names
   az keyvault secret set \
     --vault-name cloudsoftkeyvault$(date +%d%m%y) \
     --name "MongoDb--ConnectionString" \
     --value "$CONNECTION_STRING"
   ```

   > 💡 **Information**
   >
   > - Azure Key Vault uses `--` instead of `:` in secret names to match ASP.NET Core's configuration hierarchy
   > - The secret name `MongoDb--ConnectionString` maps to `MongoDb:ConnectionString` in your application

### Step 8: Test Local Development with Azure Key Vault

1. Ensure your Azure CLI is logged in:

   ```bash
   az login
   ```

2. Verify the appsettings.Development.json has the correct Key Vault URI and feature flags enabled:

   ```json
   "FeatureFlags": {
     "UseMongoDb": true,
     "UseAzureStorage": false,
     "UseAzureKeyVault": true
   },
   "AzureKeyVault": {
     "KeyVaultUri": "https://cloudsoftkeyvault250324.vault.azure.net/"
   }
   ```

3. Run the application:

   ```bash
   dotnet run
   ```

4. Check the console output for:
   - "Using Azure Key Vault for configuration"
   - "Using MongoDB repository"

5. Verify the application is connecting to Cosmos DB instead of the local MongoDB by checking logs and application behavior.

> 💡 **Information**
>
> - The connection string from Azure Key Vault will override the one in appsettings.Development.json
> - If you're seeing connection errors, check that the Key Vault URI is correct and that your Azure CLI account has access

### Phase II: Deploying to Azure VM with Managed Identity

### Step 9: Provision an Azure Virtual Machine

1. Create a cloud-init file for VM provisioning:

   > `cloud-init_dotnet.yaml`

   ```yaml
   #cloud-config

   # Update the package list
   package_update: true

   runcmd:
     # Install .Net Runtime 9.0
     - add-apt-repository ppa:dotnet/backports
     - apt-get update
     - apt-get install -y aspnetcore-runtime-9.0

     # Create the /opt/CloudSoft directory
     - mkdir -p /opt/CloudSoft
     - chown azureuser:azureuser /opt/CloudSoft

   # Create a service for the application
   write_files:
     - path: /etc/systemd/system/CloudSoft.service
       content: |
         [Unit]
         Description=ASP.NET Web App running on Ubuntu

         [Service]
         WorkingDirectory=/opt/CloudSoft
         ExecStart=/usr/bin/dotnet /opt/CloudSoft/CloudSoft.dll
         Restart=always
         RestartSec=10
         KillSignal=SIGINT
         SyslogIdentifier=CloudSoft
         User=www-data
         EnvironmentFile=/etc/CloudSoft/.env

         [Install]
         WantedBy=multi-user.target            
       owner: root:root
       permissions: '0644'

       # Create a directory for environment variables for the application
     - path: /etc/CloudSoft/.env
       content: |
         ASPNETCORE_ENVIRONMENT=Production
         ASPNETCORE_URLS="http://+:5000"      
       owner: root:root
       permissions: '0600'

   systemd:
     units:
       - name: CloudSoft.service
         enabled: true
   ```

2. Create a VM deployment script:

   > `provision_app_vm.sh`

   ```bash
   #!/bin/bash

   resource_group=CloudSoftRG
   vm_name=CloudSoftVM
   vm_port=5000

   # Create resource group
   az group create --location northeurope --name $resource_group

   # Create VM with cloud-init
   az vm create --resource-group $resource_group --name $vm_name \
                --image Ubuntu2404 --size Standard_B1s \
                --generate-ssh-keys --admin-username azureuser \
                --custom-data @cloud-init_dotnet.yaml

   # Open port
   az vm open-port --port $vm_port --resource-group $resource_group --name $vm_name

   # Get public IP
   vm_pub_ip=$(az vm show \
     --resource-group $resource_group --name $vm_name \
     --show-details --query [publicIps] --output tsv)

   # SSH into VM
   echo "You can now SSH into the VM with the following command:"
   echo "ssh azureuser@$vm_pub_ip"
   ```

3. Make the script executable and run it:

   ```bash
   chmod +x provision_app_vm.sh
   ./provision_app_vm.sh
   ```

### Step 10: Create Production Settings

1. Create an `appsettings.Production.json` file:

   > `appsettings.Production.json`

   ```json
   {
     "Logging": {
       "LogLevel": {
         "Default": "Information",
         "Microsoft.AspNetCore": "Warning"
       }
     },
     "FeatureFlags": {
       "UseMongoDb": false,
       "UseAzureStorage": false,
       "UseAzureKeyVault": false
     },
     "MongoDb": {
       "ConnectionString": "Get from Key Vault",
       "DatabaseName": "cloudsoft",
       "SubscribersCollectionName": "subscribers"
     },
     "AzureKeyVault": {
       "KeyVaultUri": "https://cloudsoftkeyvault250324.vault.azure.net/"
     }
   }
   ```

### Step 11: Create Deployment Script

1. Create a deployment script:

   > `deploy_app.sh`

   ```bash
   #!/bin/bash

   resource_group=CloudSoftRG
   vm_name=CloudSoftVM
   vm_port=5000

   # Get public IP
   vm_pub_ip=$(az vm show \
     --resource-group $resource_group --name $vm_name \
     --show-details --query [publicIps] --output tsv)

   # Publish the application
   echo "Publishing application..."
   dotnet publish ../CloudSoft.csproj --configuration Release --output ./publish

   # Stop the service before copying files
   echo "Stopping the service..."
   ssh azureuser@${vm_pub_ip} "sudo systemctl stop CloudSoft.service"

   # Copy files to VM
   echo "Copying files to VM..."
   scp -r ./publish/* azureuser@${vm_pub_ip}:/opt/CloudSoft/

   # Start service
   echo "Starting service..."
   ssh azureuser@${vm_pub_ip} "sudo systemctl start CloudSoft.service"

   # Cleanup
   rm -Rf ./publish

   # Browser URL
   echo "Deployment complete! Application is running at:"
   echo "http://$vm_pub_ip:$vm_port"
   ```

2. Make the script executable and run it to deploy the application:

   ```bash
   chmod +x deploy_app.sh
   ./deploy_app.sh
   ```

3. Verify that the application is running on the VM with the feature flags disabled:

   ```bash
   curl http://VM_IP_ADDRESS:5000
   ```

### Step 12: Configure Managed Identity for Key Vault Access

1. Enable system-assigned managed identity for the VM:

   ```bash
   az vm identity assign \
     --resource-group CloudSoftRG \
     --name CloudSoftVM
   ```

2. Get the VM's principal ID:

   ```bash
   VM_PRINCIPAL_ID=$(az vm identity show \
     --resource-group CloudSoftRG \
     --name CloudSoftVM \
     --query principalId \
     --output tsv)
   ```

3. Assign the "Key Vault Secrets User" role to the VM:

   ```bash
   az role assignment create \
     --assignee $VM_PRINCIPAL_ID \
     --role "Key Vault Secrets User" \
     --scope $(az keyvault show --name cloudsoftkeyvault250324 --resource-group CloudSoftRG --query id -o tsv)
   ```

   > 💡 **Information**
   >
   > - The "Key Vault Secrets User" role allows the VM to read secrets but not modify them
   > - This follows the principle of least privilege
   > - The scope limits access to just this specific Key Vault

### Step 13: Update Production Settings and Deploy

1. Update `appsettings.Production.json` to enable MongoDB and Azure Key Vault:

   > `appsettings.Production.json`

   ```json
   {
     "Logging": {
       "LogLevel": {
         "Default": "Information",
         "Microsoft.AspNetCore": "Warning"
       }
     },
     "FeatureFlags": {
       "UseMongoDb": true,
       "UseAzureStorage": false,
       "UseAzureKeyVault": true
     },
     "MongoDb": {
       "ConnectionString": "Get from Key Vault",
       "DatabaseName": "cloudsoft",
       "SubscribersCollectionName": "subscribers"
     },
     "AzureKeyVault": {
       "KeyVaultUri": "https://cloudsoftkeyvault250324.vault.azure.net/"
     }
   }
   ```

2. Deploy the updated application:

   ```bash
   ./deploy_app.sh
   ```

3. Verify that the application is now successfully connecting to Cosmos DB using the connection string from Key Vault:

   ```bash
   ssh azureuser@VM_IP_ADDRESS "sudo journalctl -u CloudSoft.service -n 100"
   ```

## How It Works: Configuration Provider Precedence in Detail

In ASP.NET Core, the configuration system loads values from multiple sources, with each subsequent provider potentially overriding values from previous ones. Here's the exact precedence order in our application:

1. **Default values in code** (baseline)
2. **appsettings.json** (common settings for all environments)
3. **appsettings.{Environment}.json** (environment-specific, e.g., appsettings.Production.json)
4. **Environment variables** (particularly useful for containerized applications)
5. **Command line arguments** (highest precedence for local overrides)
6. **Azure Key Vault** (specifically added in our configuration pipeline)

When our application starts:

1. First, default values are established.
2. Next, values from appsettings.json are loaded.
3. Then, environment-specific values (Development/Production) override those.
4. When Azure Key Vault is enabled, it's added after these sources.

This means secrets stored in Azure Key Vault (like our MongoDB connection string) will override the same settings defined in appsettings files, allowing us to have secure, environment-specific values without exposing them in configuration files.

The key configuration points in our Program.cs include:

```csharp
// Register Azure Key Vault after basic configuration
builder.Configuration.AddAzureKeyVault(
    new Uri(keyVaultUri),
    new DefaultAzureCredential());
```

This specific order ensures our application gets the most secure configuration possible while still maintaining flexibility.

## Managed Identity Authentication Flow

When using managed identity on the VM, the authentication flow works as follows:

1. The application uses `DefaultAzureCredential()`, which attempts multiple authentication methods in a specific order.
2. On the VM with managed identity, it accesses an Azure Instance Metadata Service endpoint on the VM.
3. This service provides an access token for the VM's identity to access Azure services.
4. Azure Key Vault accepts this token and grants access to secrets according to the assigned RBAC permissions.
5. The connection string is securely retrieved and used to connect to Cosmos DB.

This approach eliminates the need for storing credentials in the application or on the VM, significantly enhancing security.

## Final Tests

To verify everything is working correctly:

1. Check VM logs for successful startup:

   ```bash
   ssh azureuser@VM_IP_ADDRESS "sudo journalctl -u CloudSoft.service -n 100"
   ```

2. Look for these log lines:

   - "Using Azure Key Vault for configuration"
   - "Using MongoDB repository"

3. Test the application's functionality that depends on the database:
   - Navigate to the subscriber form
   - Add a new subscriber
   - Verify the subscriber is saved to Cosmos DB

4. For full verification, you can check Cosmos DB in the Azure Portal to see that data is being stored correctly.

## Troubleshooting

### Common Issues and Solutions

1. **Connection String Format Issues**
   - Cosmos DB connection strings are formatted differently from standard MongoDB strings
   - Ensure SSL is enabled and retrywrites is set correctly
   - Verify that the appName parameter is correctly set

2. **Managed Identity Problems**
   - Check that the VM has a system-assigned identity
   - Verify RBAC role assignments with `az role assignment list`
   - Ensure the VM has been restarted after enabling managed identity

3. **Key Vault Access Issues**
   - Check Key Vault firewall settings
   - Verify the Key Vault URI is correct in the application settings
   - Ensure the application has appropriate permissions

4. **Default Credential Authentication Problems**
   - In development: Verify Azure CLI login status
   - In production: Check VM's managed identity status
   - Review application logs for authentication errors

5. **Application Cannot Find Secrets**
   - Secret names in Key Vault use `--` instead of `:`
   - Verify secret names match configuration paths (e.g., `MongoDb--ConnectionString`)

### Diagnostic Commands

```bash
# Check VM managed identity
az vm identity show --resource-group CloudSoftRG --name CloudSoftVM

# Check role assignments
az role assignment list --assignee VM_PRINCIPAL_ID

# Check Key Vault access policies
az keyvault show --name cloudsoftkeyvault250324 --resource-group CloudSoftRG

# Test Key Vault access from VM
ssh azureuser@VM_IP_ADDRESS "curl 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https%3A%2F%2Fvault.azure.net' -H 'Metadata: true'"
```

## Done! 🎉

Congratulations! You've successfully implemented Azure Key Vault integration with your ASP.NET Core application, deployed it to an Azure VM using managed identity, and connected securely to Cosmos DB. This secure architecture ensures your application can access sensitive configuration without exposing credentials in your code or configuration files.

### Key Concepts Learned

- **Configuration Provider System** in ASP.NET Core
- **Azure Key Vault** for secure secret management
- **Managed Identity** for passwordless authentication
- **Options Pattern** for strongly-typed configuration
- **Feature Flags** for environment-specific behavior

This pattern can be extended to secure any sensitive configuration data your application needs, such as API keys, connection strings, and other secrets. 🚀
