+++
title = "2. Nginx Deployment with Ansible"
weight = 2
date = 2025-03-17
draft = false
+++

# Exercise 2: Nginx Deployment with Ansible

## Goal

Create an Ansible role to install and configure Nginx with a custom welcome page, learning how to use templates, variables, and handlers to manage web server configurations.

## Learning Objectives

By the end of this exercise, you will:

- Create a complete **Ansible role structure** with proper organization
- Use the **apt module** to install packages
- Configure **service management** to control Nginx
- Implement **Jinja2 templates** for dynamic HTML and configuration files
- Configure **Nginx settings** including custom port assignment
- Understand **handlers** for service restart management
- Test and **validate your deployment** with HTTP requests

## Prerequisites

- Completed Exercise 1 with a working Ansible installation
- Basic familiarity with web servers and HTTP
- Access to your Azure VM provisioned via SSH

## Step-by-Step Instructions

### Step 1: Create the Nginx Role Structure

First, let's create the directory structure for our Nginx role:

```bash
# Make sure you're in your ansible-learning directory
cd ~/ansible-learning

# Create the role directories
mkdir -p roles/nginx_hello_world/{tasks,handlers,defaults,templates,meta}
```

> 💡 **Information**
>
> - Ansible roles follow a standard directory structure
> - `tasks` contain the main actions to perform
> - `handlers` define actions that respond to notifications (like service restarts)
> - `defaults` contain default variable values
> - `templates` store Jinja2 template files
> - `meta` defines role metadata including dependencies

### Step 2: Create the Base Nginx Role

Let's first create a basic role for installing Nginx:

1. Create the directory structure for the base nginx role:

    ```bash
    mkdir -p roles/nginx/{tasks,handlers}
    ```

2. Create the main tasks file for installing Nginx:

    ```bash
    cat > roles/nginx/tasks/main.yaml << 'EOF'
    ---
    - name: Install Nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Ensure Nginx is running and enabled
      service:
        name: nginx
        state: started
        enabled: yes
    EOF
    ```

3. Create a handler for restarting Nginx:

    ```bash
    cat > roles/nginx/handlers/main.yaml << 'EOF'
    ---
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
    EOF
    ```

> 💡 **Information**
>
> - The `apt` module manages packages on Debian-based systems
> - The `service` module controls system services
> - `update_cache: yes` is equivalent to running `apt update`
> - `enabled: yes` ensures the service starts automatically after reboot

### Step 3: Create the Hello World Website Role

Now, let's create our role for the custom Nginx site:

1. Create the main tasks file for the Hello World site:

    ```bash
    cat > roles/nginx_hello_world/tasks/main.yaml << 'EOF'
    ---
    - name: Remove default nginx site
      file:
        path: "{{ item }}"
        state: absent
      with_items:
        - /etc/nginx/sites-enabled/default
        - /etc/nginx/sites-available/default
      notify: Restart nginx

    - name: Create new hello-world site
      template:
        src: hello-world.conf.j2
        dest: /etc/nginx/sites-available/hello-world.conf
        owner: root
        group: root
        mode: '0644'
      notify: Restart nginx

    - name: Enable the site
      file:
        src: /etc/nginx/sites-available/hello-world.conf
        dest: /etc/nginx/sites-enabled/hello-world.conf
        state: link
      notify: Restart nginx

    - name: Create web root directory
      file:
        path: /var/www/html
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'

    - name: Deploy the index.html template
      template:
        src: index.html.j2
        dest: /var/www/html/index.html
        owner: www-data
        group: www-data
        mode: '0644'
      notify: Restart nginx

    - name: Deploy the CSS file
      template:
        src: style.css.j2
        dest: /var/www/html/style.css
        owner: www-data
        group: www-data
        mode: '0644'
    EOF
    ```

2. Create the handler file:

    ```bash
    cat > roles/nginx_hello_world/handlers/main.yaml << 'EOF'
    ---
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
    EOF
    ```

3. Create defaults file to define variables:

    ```bash
    cat > roles/nginx_hello_world/defaults/main.yaml << 'EOF'
    ---
    nginx_port: 8080
    site_title: "Hello from Nginx!"
    site_heading: "Hello, World!"
    site_message: "This page is served by Nginx on {{ ansible_hostname }}"
    site_footer: "Deployed using Ansible"
    site_color: "#4285F4"
    EOF
    ```

4. Create the site configuration template:

    ```bash
    cat > roles/nginx_hello_world/templates/hello-world.conf.j2 << 'EOF'
    server {
        listen {{ nginx_port }};
        server_name _;

        root /var/www/html;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }
        
        # Deny access to hidden files
        location ~ /\. {
            deny all;
        }
        
        # Custom error pages
        error_page 404 /404.html;
        error_page 500 502 503 504 /50x.html;
    }
    EOF
    ```

5. Create HTML template:

    ```bash
    cat > roles/nginx_hello_world/templates/index.html.j2 << 'EOF'
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>{{ site_title }}</title>
        <link rel="stylesheet" href="style.css">
    </head>
    <body>
        <div class="container">
            <header>
                <h1>{{ site_heading }}</h1>
            </header>
            
            <main>
                <p>{{ site_message }}</p>
                <p>Server Information:</p>
                <ul>
                    <li>Hostname: {{ ansible_hostname }}</li>
                    <li>IP Address: {{ ansible_default_ipv4.address }}</li>
                    <li>OS: {{ ansible_distribution }} {{ ansible_distribution_version }}</li>
                    <li>Current Time: {{ ansible_date_time.iso8601 }}</li>
                </ul>
            </main>
            
            <footer>
                <p>{{ site_footer }}</p>
            </footer>
        </div>
    </body>
    </html>
    EOF
    ```

6. Create CSS template:

    ```bash
    cat > roles/nginx_hello_world/templates/style.css.j2 << 'EOF'
    /* Basic style for Hello World page */
    body {
        font-family: Arial, sans-serif;
        line-height: 1.6;
        margin: 0;
        padding: 0;
        background-color: #f4f4f4;
        color: #333;
    }

    .container {
        width: 80%;
        max-width: 800px;
        margin: 2rem auto;
        padding: 2rem;
        background-color: white;
        border-radius: 8px;
        box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
    }

    header {
        text-align: center;
        margin-bottom: 2rem;
    }

    h1 {
        color: {{ site_color }};
        border-bottom: 2px solid {{ site_color }};
        padding-bottom: 0.5rem;
    }

    ul {
        background-color: #f9f9f9;
        padding: 1rem 1rem 1rem 2rem;
        border-radius: 4px;
    }

    li {
        margin-bottom: 0.5rem;
    }

    footer {
        margin-top: 2rem;
        text-align: center;
        font-size: 0.9rem;
        color: #666;
        border-top: 1px solid #eee;
        padding-top: 1rem;
    }
    EOF
    ```

7. Create role dependency on the base Nginx role:

    ```bash
    cat > roles/nginx_hello_world/meta/main.yml << 'EOF'
    ---
    dependencies:
      - role: nginx
    EOF
    ```

> 💡 **Information**
>
> - The `template` module uses Jinja2 templates to generate files
> - Variables like `{{ nginx_port }}` are replaced with values
> - The `file` module creates symbolic links to enable the site
> - `notify: Restart nginx` triggers the handler when a task changes
> - The `meta/main.yml` file defines dependencies on other roles
> - The `.j2` extension is the convention for Jinja2 templates

### Step 4: Create a Playbook for the Hello World Website

Let's create a playbook to apply our Nginx role:

```bash
cat > hello_world_playbook.yml << 'EOF'
---
- name: Deploy Hello World Website
  hosts: web
  become: yes
  
  roles:
    - role: nginx_hello_world
      vars:
        site_title: "Hello from Ansible Automation!"
        site_heading: "Hello, Azure VM!"
        site_message: "This is a demo website deployed via Ansible roles"
        site_color: "#34A853"
        nginx_port: 8080
EOF
```

### Step 5: Update the Inventory File

Make sure your inventory file points to your Azure VM:

```bash
cat > inventory.yaml << 'EOF'
all:
  hosts:
    web:
      ansible_host: YOUR_VM_IP_ADDRESS
  vars:
    ansible_ssh_common_args: '-o StrictHostKeyChecking=no'
    ansible_python_interpreter: /usr/bin/python3
    ansible_become: yes
    ansible_user: azureuser
EOF
```

> 💡 **Information**
>
> - Replace `YOUR_VM_IP_ADDRESS` with the actual IP of your Azure VM
> - `ansible_become: yes` enables privilege escalation (sudo)
> - `ansible_user: azureuser` specifies the SSH user

### Step 6: Provision an Azure VM

Ensure your Azure VM allows traffic on port 8080:

```bash
#!/bin/bash

# Variables
RESOURCE_GROUP=AnsibleDemoRG
PORTS=(80 443 8080 5000)  # Array of ports to open
VM_NAME=DemoVM

# Create resource group
az group create --name $RESOURCE_GROUP --location northeurope

# Create VM with cloud-init
az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --image Ubuntu2404 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys

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

# Generate new inventory file
echo "Updating inventory file..."
cat > inventory.yaml << EOF
all:
  hosts:
    web:
      ansible_host: ${IP}
  vars:
    ansible_ssh_common_args: '-o StrictHostKeyChecking=no'
    ansible_python_interpreter: /usr/bin/python3
    ansible_become: yes
    ansible_user: azureuser
EOF

# SSH into VM
echo "You can now SSH into the VM with the following command:"
echo "ssh azureuser@$IP"
```

### Step 7: Deploy the Website

Run the playbook to deploy your Hello World website:

```bash
ansible-playbook hello_world_playbook.yml
```

### Step 8: Test the Website

Verify that the website is running correctly:

```bash
VM_IP=$(sed -n 's/.*ansible_host: \([0-9.]*\).*/\1/p' inventory.yaml)
curl -s http://${VM_IP}:8080
```

You can also visit the website in your browser at `http://<VM_IP>:8080`.

### Step 9: Modify the Website

Let's update our playbook to change the website appearance:

```bash
cat > hello_world_update.yml << 'EOF'
---
- name: Deploy Hello World Website
  hosts: web
  become: yes
  
  roles:
    - role: nginx_hello_world
      vars:
        site_title: "Updated Nginx Site"
        site_heading: "Hello from Ansible!"
        site_message: "This page has been updated through Ansible automation"
        site_color: "#DB4437"
        nginx_port: 8080
EOF
```

Deploy the updated website:

```bash
ansible-playbook hello_world_update.yml
```

Check the website again to see your changes.

## Troubleshooting

If you encounter issues during this exercise, here are some common solutions:

### Nginx Installation Issues

- Check if the package repository is accessible
- Verify that you have sufficient permissions (sudo/become)
- Look at logs with `ansible web -i inventory.yaml -m command -a "tail /var/log/apt/term.log"`

### Website Not Accessible

- Verify Azure NSG allows traffic on port 8080
- Check Nginx is running: `ansible web -i inventory.yaml -m command -a "systemctl status nginx"`
- Check the Nginx configuration: `ansible web -i inventory.yaml -m command -a "nginx -t"`
- Look at error logs: `ansible web -i inventory.yaml -m command -a "tail /var/log/nginx/error.log"`

### Template Errors

- Verify syntax in your Jinja2 templates
- Check file paths and permissions
- Use verbose mode to see details: `ansible-playbook hello_world_playbook.yml -i inventory.yaml -v`

### Role Dependency Issues

- Ensure the `meta/main.yml` file has correct formatting
- Check that both roles are in the correct directories
- Verify role naming is consistent

## Final Tests

To verify that you've completed the exercise successfully:

1. Check if Nginx is running:

   ```bash
   ansible web -i inventory.yaml -m command -a "systemctl status nginx"
   ```

2. Verify the website configuration:

   ```bash
   ansible web -i inventory.yaml -m command -a "cat /etc/nginx/sites-enabled/hello-world.conf"
   ```

3. Verify the HTML content:

   ```bash
   ansible web -i inventory.yaml -m command -a "cat /var/www/html/index.html"
   ```

4. Test HTTP access to your website:

    ```bash
    VM_IP=$(sed -n 's/.*ansible_host: \([0-9.]*\).*/\1/p' inventory.yaml)
    curl -s http://${VM_IP}:8080
    ```

5. Check Nginx configuration syntax:

   ```bash
   ansible web -i inventory.yaml -m command -a "nginx -t"
   ```

✅ **Expected Results**

- Nginx is installed and running
- The default site is disabled
- Your custom site is configured on port 8080
- The HTML page displays your custom title, heading, and message
- The page styling matches your configuration

## Done! 🎉

Congratulations! You've successfully deployed a custom Nginx website using Ansible roles. You've learned how to:

- Create and organize Ansible roles
- Use variables to make templates dynamic
- Work with handlers for service management
- Configure Nginx to serve a custom website
- Deploy and update configurations using playbooks

In the next exercise, we'll build on this foundation by setting up a .NET runtime environment that will prepare our server for hosting .NET applications.
