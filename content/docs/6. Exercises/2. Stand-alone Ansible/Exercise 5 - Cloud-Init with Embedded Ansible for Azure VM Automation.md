+++
title = "5. Cloud-Init with Embedded Ansible for Azure VM Automation"
weight = 5
date = 2025-03-19
draft = false
+++

# Exercise 5: Cloud-Init with Embedded Ansible for Azure VM Automation

## Goal

Implement an automated deployment solution using cloud-init with embedded Ansible playbooks to provision an Azure VM that hosts a .NET application behind a secure Nginx reverse proxy, demonstrating how Ansible can be used as a standalone automation tool within cloud-init.

## Learning Objectives

By the end of this exercise, you will:

- Create **cloud-init configurations** with embedded Ansible content
- Use **cloud-init as a bootstrapping mechanism** for VM provisioning
- Implement **embedded Ansible playbooks and roles** within cloud-init
- Configure an **Azure VM** using the custom cloud-init script
- Understand the **cloud-init execution flow** with Ansible integration
- Deploy a **.NET application** to an automatically configured environment
- Compare **different automation approaches** (standalone Ansible vs. cloud-init embedded Ansible)

## Prerequisites

- Completed Exercises 1-4, or at least familiar with the concepts
- Azure CLI installed and configured
- A .NET MVC web application named CloudSoft in a parallel directory
- Basic understanding of cloud-init concepts

## Step-by-Step Instructions

### Step 1: Understanding Cloud-Init with Embedded Ansible

Before we start creating our files, let's understand the structure and benefits of this approach:

```
Cloud-Init with Embedded Ansible
└── cloud-init_ansible.yaml
    ├── package_update & installation
    ├── runcmd (execute Ansible)
    └── write_files
        ├── Ansible inventory
        ├── Ansible playbook
        └── Ansible roles
            ├── dotnet_cloudsoft role
            └── nginx_reverse_proxy role
```

> 💡 **Information**
>
> - **Cloud-init** is a utility for early initialization of cloud instances
> - It executes during the first boot of a VM
> - By embedding Ansible in cloud-init, we eliminate the need for external Ansible control nodes
> - This approach combines the benefits of declarative configurations (Ansible) with bootstrapping (cloud-init)
> - The VM is fully configured on first boot without additional access required

### Step 2: Create the Cloud-Init Configuration File

Let's create a cloud-init file with embedded Ansible configuration:

> cloud-init_ansible.yaml

```yaml
#cloud-config

package_update: true

packages:
  - python3
  - ansible

runcmd:
  - ansible-playbook /opt/ansible/playbook.yaml

write_files:
  - path: /etc/ansible/hosts
    content: |
      [local]
      localhost ansible_connection=local
    owner: root:root
    permissions: '0644'

  - path: /opt/ansible/playbook.yaml
    content: |
      ---
      - name: Execute Ansible playbook
        hosts: local
        become: yes
        roles:
          - dotnet_cloudsoft
          - nginx_reverse_proxy
    owner: root:root
    permissions: '0644'

  - path: /opt/ansible/roles/dotnet_cloudsoft/tasks/main.yml
    content: |
      ---
      - name: Add .NET repository
        apt_repository:
          repo: ppa:dotnet/backports
          state: present

      - name: Install .NET Runtime 9.0
        apt:
          name: aspnetcore-runtime-9.0
          state: present
          update_cache: yes

      - name: Create CloudSoft directory
        file:
          path: /opt/CloudSoft
          state: directory
          owner: azureuser
          group: azureuser
          mode: '0755'

      - name: Create environment file directory
        file:
          path: /etc/CloudSoft
          state: directory
          mode: '0755'

      - name: Create environment file
        copy:
          dest: /etc/CloudSoft/.env
          content: |
            ASPNETCORE_ENVIRONMENT=Production
            ASPNETCORE_URLS="http://+:5000"
          mode: '0600'
          owner: root
          group: root

      - name: Create systemd service file
        copy:
          dest: /etc/systemd/system/CloudSoft.service
          content: |
            [Unit]
            Description=ASP.NET Web App running on Ubuntu
            After=nginx.service

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
          mode: '0644'
          owner: root
          group: root

      - name: Reload systemd and enable CloudSoft service
        systemd:
          name: CloudSoft
          enabled: yes
          state: started
          daemon_reload: yes
    owner: root:root
    permissions: '0644'

  - path: /opt/ansible/roles/nginx_reverse_proxy/tasks/main.yml
    content: |
      ---
      - name: Install Nginx
        apt:
          name: nginx
          state: present

      - name: Generate SSL certificate (ensures directory exists)
        shell: |
          mkdir -p /etc/nginx/ssl
          if [ ! -f /etc/nginx/ssl/nginx.key ] || [ ! -f /etc/nginx/ssl/nginx.crt ]; then
            openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
              -keyout /etc/nginx/ssl/nginx.key \
              -out /etc/nginx/ssl/nginx.crt \
              -subj "/C=SE/ST=VastraGotaland/L=Molndal/O=CloudSoft/CN=localhost"
          fi
        args:
          creates: /etc/nginx/ssl/nginx.crt

      - name: Create Nginx configuration
        copy:
          dest: /etc/nginx/sites-available/cloudsoft
          content: |
            server {
                listen 80;
                server_name _;
                return 301 https://$host$request_uri;
            }

            server {
                listen 443 ssl;
                server_name _;

                ssl_certificate /etc/nginx/ssl/nginx.crt;
                ssl_certificate_key /etc/nginx/ssl/nginx.key;
                
                ssl_protocols TLSv1.2 TLSv1.3;
                ssl_prefer_server_ciphers on;
                ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;

                location / {
                    proxy_pass http://localhost:5000;
                    proxy_http_version 1.1;
                    proxy_set_header Upgrade $http_upgrade;
                    proxy_set_header Connection keep-alive;
                    proxy_set_header Host $host;
                    proxy_cache_bypass $http_upgrade;
                    proxy_set_header X-Real-IP $remote_addr;
                    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_set_header X-Forwarded-Proto $scheme;
                }
            }
          mode: '0644'

      - name: Enable site configuration
        file:
          src: /etc/nginx/sites-available/cloudsoft
          dest: /etc/nginx/sites-enabled/cloudsoft
          state: link

      - name: Remove default nginx site
        file:
          path: /etc/nginx/sites-enabled/default
          state: absent

      - name: Test Nginx configuration
        command: nginx -t
        register: nginx_test
        changed_when: nginx_test.rc != 0

      - name: Restart Nginx if configuration is valid
        systemd:
          name: nginx
          state: restarted
          enabled: yes
        when: nginx_test.rc == 0
```

> 💡 **Information**
>
> - The `#cloud-config` line is mandatory and must be the first line
> - `package_update: true` ensures the package list is updated before installation
> - `packages` section installs Python3 and Ansible
> - `runcmd` section runs the Ansible playbook after all files are written
> - `write_files` section creates all necessary Ansible files with their content
> - The Ansible configuration manages both the .NET environment and Nginx reverse proxy

### Step 3: Create the VM Provisioning Script

Now, let's create a script to provision an Azure VM with our cloud-init configuration:

> provision_vm.sh

```bash
#!/bin/bash

# Variables
RESOURCE_GROUP=CloudSoftBehindReverseProxyWithAnsibleRG
CUSTOM_DATA_FILE="cloud-init_ansible.yaml"
PORTS=(80 443 5000)  # Array of ports to open
VM_NAME=CloudSoftVM

# Create resource group
az group create --name $RESOURCE_GROUP --location northeurope

# Create VM with cloud-init
az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --image Ubuntu2404 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --custom-data @$CUSTOM_DATA_FILE

# Open ports with incrementing priority
PRIORITY=1100
for PORT in "${PORTS[@]}"; do
  echo "Opening port $PORT with priority $PRIORITY..."
  az vm open-port \
    --resource-group $RESOURCE_GROUP \
    --name $VM_NAME \
    --port $PORT \
    --priority $PRIORITY
  ((PRIORITY+=100))
done

# Get public IP
IP=$(az vm show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --show-details \
  --query [publicIps] \
  --output tsv)

# SSH into VM
echo "You can now SSH into the VM with the following command:"
echo "ssh azureuser@$IP"
```

> 💡 **Information**
>
> - The `--custom-data @$CUSTOM_DATA_FILE` parameter passes our cloud-init file
> - `@` prefix tells Azure CLI to read the file content
> - The script opens ports 80, 443, and 5000 on the Azure NSG
> - SSH keys are automatically generated with `--generate-ssh-keys`
> - The script outputs the SSH command for easy access

### Step 4: Create the Application Deployment Script

Let's create a script to deploy our .NET application to the provisioned VM:

> deploy_app.sh

```bash
#!/bin/bash

# Get VM public IP
VM_IP=$(az vm show --show-details \
    --resource-group CloudSoftBehindReverseProxyWithAnsibleRG \
    --name CloudSoftVM \
    --query [publicIps] --output tsv)

# Publish the application
echo "Publishing application..."
dotnet publish ../src/CloudSoft --configuration Release --output ./publish

# Stop the service before copying files
echo "Stopping the service..."
ssh azureuser@${VM_IP} "sudo systemctl stop CloudSoft.service"

# Copy files to VM
echo "Copying files to VM..."
scp -r ./publish/* azureuser@${VM_IP}:/opt/CloudSoft/

# Start service
echo "Starting service..."
ssh azureuser@${VM_IP} "sudo systemctl start CloudSoft.service"

# Cleanup
rm -Rf ./publish

# Browser URL
echo "Deployment complete! Application is running at:"
echo "https://${VM_IP}"
```

> 💡 **Information**
>
> - The script assumes the CloudSoft app is in a parallel directory (`../../CloudSoft`)
> - It stops the service before copying new files to ensure a clean deployment
> - The application files are copied directly to the `/opt/CloudSoft/` directory
> - After deployment, it displays the HTTPS URL for accessing the application

### Step 5: Execute the Provisioning Process

Now let's run the provisioning script to create our VM:

```bash
./provision_vm.sh
```

This will:
1. Create a new resource group in Azure
2. Provision a VM with Ubuntu 24.04
3. Apply our cloud-init configuration which will:
   - Install Python3 and Ansible
   - Write all Ansible files to the VM
   - Execute the Ansible playbook automatically
4. Configure the Azure Network Security Group
5. Output the VM's public IP address

> 💡 **Information**
>
> - The VM provisioning may take 5-10 minutes to complete
> - Cloud-init execution happens during the first boot
> - You can monitor cloud-init progress by connecting to the VM and checking `/var/log/cloud-init-output.log`

### Step 6: Monitor Cloud-Init and Ansible Execution

After the VM is provisioned, let's connect to it and check the status of cloud-init and Ansible:

```bash
# Get the VM's IP address
VM_IP=$(az vm show --show-details \
    --resource-group CloudSoftBehindReverseProxyWithAnsibleRG \
    --name CloudSoftVM \
    --query [publicIps] --output tsv)

# SSH into the VM
ssh azureuser@${VM_IP}

# Check cloud-init logs
sudo cat /var/log/cloud-init-output.log

# Check Ansible playbook logs
sudo cat /var/log/cloud-init.log | grep ansible

# Verify that the services are running
systemctl status nginx
systemctl status CloudSoft
```

### Step 7: Deploy the .NET Application

Once the VM is fully provisioned and configured, we can deploy our .NET application:

```bash
./deploy_app.sh
```

After the deployment completes, you should be able to access your application at:
```
https://<your-vm-ip>
```

> 💡 **Information**
>
> - Since we're using a self-signed SSL certificate, your browser will show a security warning
> - You can proceed past the warning to see your application
> - The application is running behind the Nginx reverse proxy with SSL termination

### Step 8: Understanding the Benefits of This Approach

Let's compare this approach to the previous exercises:

1. **Traditional Ansible (Exercises 1-4):**
   - Requires a control node with Ansible installed
   - Requires SSH access to target VMs
   - Configuration happens after VM provisioning
   - Changes or updates need another Ansible run

2. **Cloud-Init with Embedded Ansible (This Exercise):**
   - No separate control node required
   - Configuration happens during first boot
   - VM is fully configured before anyone logs in
   - Perfect for immutable infrastructure or scaling scenarios
   - Self-contained, no external dependencies once VM is deployed

## Troubleshooting

If you encounter issues during this exercise, here are some common solutions:

### Cloud-Init Execution Issues
- Check cloud-init logs: `sudo cat /var/log/cloud-init-output.log`
- Verify cloud-init status: `cloud-init status`
- Check for syntax errors in YAML: validate your cloud-init file at [yamllint.com](https://www.yamllint.com/)

### Ansible Execution Issues
- Check if Ansible was installed: `ansible --version`
- Look for errors in the Ansible execution: `grep -A10 ansible /var/log/cloud-init-output.log`
- Check if files were created correctly: `ls -la /opt/ansible/`

### Application Deployment Issues
- Verify directory permissions: `ls -la /opt/CloudSoft/`
- Check service status: `systemctl status CloudSoft`
- Look at service logs: `journalctl -u CloudSoft`
- Verify Nginx configuration: `nginx -t`

### SSL/HTTPS Access Issues
- Check if certificates were generated: `ls -la /etc/nginx/ssl/`
- Verify Nginx configuration: `cat /etc/nginx/sites-enabled/cloudsoft`
- Check Nginx status: `systemctl status nginx`
- Verify open ports in Azure NSG: `az network nsg rule list --resource-group CloudSoftBehindReverseProxyWithAnsibleRG`

## Final Tests

To verify that you've completed the exercise successfully:

1. Check that the VM was provisioned correctly in Azure:
   ```bash
   az vm show \
     --resource-group CloudSoftBehindReverseProxyWithAnsibleRG \
     --name CloudSoftVM \
     --query provisioningState
   ```
   You should see `"Succeeded"`.

2. SSH into the VM and check services:
   ```bash
   VM_IP=$(az vm show --show-details \
     --resource-group CloudSoftBehindReverseProxyWithAnsibleRG \
     --name CloudSoftVM \
     --query [publicIps] --output tsv)
   
   ssh azureuser@${VM_IP} "systemctl status nginx && systemctl status CloudSoft"
   ```
   Both services should be `active (running)`.

3. Verify HTTP to HTTPS redirection:
   ```bash
   curl -I http://${VM_IP}
   ```
   You should see a 301 redirect to HTTPS.

4. Check HTTPS access:
   ```bash
   curl -k https://${VM_IP}
   ```
   You should see the HTML content of your application.

5. Access the application in a browser:
   Open `https://<your-vm-ip>` in a browser. After accepting the self-signed certificate warning, you should see your CloudSoft application.

✅ **Expected Results**

- Azure VM is provisioned with cloud-init
- Ansible playbooks and roles are deployed and executed automatically
- Nginx is configured as a reverse proxy with SSL
- .NET runtime environment is set up
- Your CloudSoft application is deployed and running
- The application is accessible via HTTPS

## Done! 🎉

Congratulations! You've successfully deployed a .NET application to an Azure VM using cloud-init with embedded Ansible. This approach demonstrates how to use Ansible as a standalone automation tool within cloud-init, combining the benefits of both technologies.

Key takeaways from this exercise:
- Cloud-init is powerful for bootstrapping VM configurations
- Ansible can be embedded within cloud-init for complex configurations
- This approach creates self-contained, fully configured VMs on first boot
- The combination is perfect for immutable infrastructure patterns
- No external Ansible control node is required

You can now use this technique for other deployment scenarios or extend it with additional Ansible roles and configuration.
