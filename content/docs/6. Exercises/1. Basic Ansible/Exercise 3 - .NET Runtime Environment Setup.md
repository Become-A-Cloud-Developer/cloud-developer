+++
title = "3. .NET Runtime Environment Setup"
weight = 3
date = 2025-03-17
draft = false
+++

# Exercise 3: .NET Runtime Environment Setup

## Goal

Set up a complete environment for hosting .NET Core applications on your Azure VM, installing the necessary runtime components and configuring the server to run .NET applications as services.

## Learning Objectives

By the end of this exercise, you will:

- Create a role to install **.NET Core runtime** dependencies
- Configure **environment files** for application settings
- Set up **systemd service templates** for application management
- Create **directory structures** with proper permissions
- Implement **role dependencies** between related components
- Understand **environment variables** in .NET applications
- Test the **.NET runtime environment** for proper configuration

## Prerequisites

- Completed Exercises 1-2
- Access to your Azure VM
- Basic understanding of web application hosting concepts

## Step-by-Step Instructions

### Step 1: Create the .NET Runtime Role

First, let's create a role to install the .NET Core runtime:

1. Create the role directory structure:

    ```bash
    # Make sure you're in your ansible-learning directory
    cd ~/ansible-learning

    # Create the role directories
    mkdir -p roles/dotnet/{tasks,defaults,meta}
    ```

2. Create the main tasks file for installing .NET:

    ```bash
    cat > roles/dotnet/tasks/main.yaml << 'EOF'
    ---
    - name: Add .NET repository
      apt_repository:
        repo: ppa:dotnet/backports
        state: present

    - name: Install .NET Runtime
      apt:
        name: aspnetcore-runtime-{{ dotnet_version }}
        state: present
        update_cache: yes

    - name: Check .NET version
      command: dotnet --info
      register: dotnet_info
      changed_when: false

    - name: Display installed .NET version
      debug:
        var: dotnet_info.stdout_lines
    EOF
    ```

3. Create default variables for the .NET role:

    ```bash
    cat > roles/dotnet/defaults/main.yaml << 'EOF'
    ---
    dotnet_version: "9.0"
    EOF
    ```

> 💡 **Information**
>
> - The `apt_key` module adds package signing keys
> - The `apt_repository` module configures package repositories
> - We use `command` to detect the Ubuntu version for the correct repository
> - `register` captures command output for later use
> - `changed_when: false` ensures the command doesn't trigger change events

### Step 2: Create the .NET Web Application Role

Now, let's create a role to set up the environment for running .NET web applications:

1. Create the directory structure:

    ```bash
    mkdir -p roles/dotnet_webapp/{tasks,handlers,defaults,templates,meta}
    ```

2. Create the main tasks file:

    ```bash
    cat > roles/dotnet_webapp/tasks/main.yaml << 'EOF'
    ---
    - name: Create application directory
      file:
        path: "/opt/{{ app_name }}"
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_user }}"
        mode: '0755'

    - name: Create environment file directory
      file:
        path: "/etc/{{ app_name }}"
        state: directory
        mode: '0755'

    - name: Create logs directory
      file:
        path: "/var/log/{{ app_name }}"
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_user }}"
        mode: '0755'

    - name: Create environment file
      template:
        src: app.env.j2
        dest: "/etc/{{ app_name }}/.env"
        mode: '0600'
        owner: root
        group: root
      notify: Restart {{ app_name }}

    - name: Create systemd service file
      template:
        src: app.service.j2
        dest: "/etc/systemd/system/{{ app_name }}.service"
        mode: '0644'
        owner: root
        group: root
      register: service_file

    - name: Reload systemd if service file changed
      systemd:
        daemon_reload: yes
      when: service_file.changed

    - name: Enable application service
      systemd:
        name: "{{ app_name }}"
        enabled: yes
      when: service_file.changed
    EOF
    ```

3. Create a handler for restarting the application:

    ```bash
    cat > roles/dotnet_webapp/handlers/main.yaml << 'EOF'
    ---
    - name: "Restart {{ app_name }}"
      systemd:
        name: "{{ app_name }}"
        state: restarted
        daemon_reload: yes
    EOF
    ```

4. Create default variables:

    ```bash
    cat > roles/dotnet_webapp/defaults/main.yaml << 'EOF'
    ---
    # Application settings
    app_name: CloudSoft
    app_user: azureuser
    app_port: 5000
    app_environment: Production
    app_bind_address: localhost

    # Feature flags
    feature_flag_use_mongodb: false
    feature_flag_use_azure_storage: false

    # Connection strings (examples)
    mongodb_connection_string: "mongodb://{username}:{password}@{hostname}:{port}"
    azure_blob_container_url: "https://{accountname}.blob.core.windows.net/{container}"

    # Logging settings
    logging_level_default: "Information"
    logging_level_microsoft: "Warning"
    logging_level_system: "Warning"

    # Application-specific settings
    app_settings: {}
    EOF
    ```

5. Create the environment file template:

    ```bash
    cat > roles/dotnet_webapp/templates/app.env.j2 << 'EOF'
    ASPNETCORE_ENVIRONMENT={{ app_environment }}
    ASPNETCORE_URLS="http://{{ app_bind_address }}:{{ app_port }}"

    # Feature flags
    FeatureFlags__UseMongoDb={{ feature_flag_use_mongodb | lower }}
    FeatureFlags__UseAzureStorage={{ feature_flag_use_azure_storage | lower }}

    # Database connections (if enabled)
    {% if feature_flag_use_mongodb %}
    MongoDb__ConnectionString="{{ mongodb_connection_string }}"
    {% endif %}

    # Cloud storage (if enabled)
    {% if feature_flag_use_azure_storage %}
    AzureBlob__ContainerUrl="{{ azure_blob_container_url }}"
    {% endif %}

    # Logging configuration
    Logging__LogLevel__Default={{ logging_level_default }}
    Logging__LogLevel__Microsoft={{ logging_level_microsoft }}
    Logging__LogLevel__System={{ logging_level_system }}

    # Application-specific settings
    {% for key, value in app_settings.items() %}
    {{ key }}={{ value }}
    {% endfor %}
    EOF
    ```

6. Create the systemd service template:

    ```bash
    cat > roles/dotnet_webapp/templates/app.service.j2 << 'EOF'
    [Unit]
    Description=ASP.NET Web App - {{ app_name }}
    After=network.target

    [Service]
    WorkingDirectory=/opt/{{ app_name }}
    ExecStart=/usr/bin/dotnet /opt/{{ app_name }}/{{ app_name }}.dll
    Restart=always
    RestartSec=10
    KillSignal=SIGINT
    SyslogIdentifier={{ app_name }}
    User={{ app_user }}
    EnvironmentFile=/etc/{{ app_name }}/.env

    # Set file system rights
    AmbientCapabilities=CAP_NET_BIND_SERVICE
    NoNewPrivileges=true
    PrivateTmp=true
    ProtectSystem=full

    # Configure stderr/stdout logging
    StandardOutput=append:/var/log/{{ app_name }}/stdout.log
    StandardError=append:/var/log/{{ app_name }}/stderr.log

    [Install]
    WantedBy=multi-user.target
    EOF
    ```

7. Create role dependencies by creating a meta file:

    ```bash
    cat > roles/dotnet_webapp/meta/main.yml << 'EOF'
    ---
    dependencies:
      - role: dotnet
    EOF
    ```

> 💡 **Information**
>
> - The systemd service file configures how the application runs as a service
> - Environment variables are loaded from the `.env` file
> - The `EnvironmentFile` directive tells systemd where to find environment variables
> - `StandardOutput/StandardError` directives configure logging
> - Role dependencies ensure that .NET is installed before configuring the app

### Step 3: Create a Playbook for .NET Environment Setup

Now let's create a playbook to set up our .NET environment:

```bash
cat > dotnet_setup.yml << 'EOF'
---
- name: Set Up .NET Environment
  hosts: web
  become: yes
  
  roles:
    - role: dotnet_webapp
      vars:
        app_name: CloudSoft
        app_port: 5000
        app_environment: Production
        app_settings:
          CUSTOM_WELCOME_MESSAGE: "Hello from Azure VM!"
          DEPLOYMENT_ID: "{{ ansible_date_time.iso8601 }}"
EOF
```

### Step 4: Run the Playbook

Execute the playbook to set up the .NET environment:

```bash
ansible-playbook dotnet_setup.yml
```

### Step 5: Verify the Installation

Let's check that the .NET runtime is installed correctly:

```bash
ansible web -m command -a "dotnet --info"
```

And check that the service configuration is in place:

```bash
ansible web -m command -a "systemctl status CloudSoft"
```

You should see that the service is loaded but not running yet (since we haven't deployed the actual application).

### Step 6: Create a Test Application Placeholder

Let's create a placeholder for our application to verify the service configuration:

```bash
cat > test_app_deploy.yml << 'EOF'
---
- name: Deploy Test Application
  hosts: web
  become: yes
  
  vars:
    app_name: CloudSoft
    
  tasks:
    - name: Create a simple test application file
      copy:
        content: |
          <!DOCTYPE html>
          <html>
          <head>
              <title>Test .NET Application</title>
          </head>
          <body>
              <h1>Test .NET Application</h1>
              <p>This is a placeholder for a real .NET application.</p>
              <p>Environment: {{ lookup('env', 'ASPNETCORE_ENVIRONMENT') }}</p>
              <p>Deployed at: {{ ansible_date_time.iso8601 }}</p>
          </body>
          </html>
        dest: "/opt/{{ app_name }}/index.html"
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        mode: '0644'
      
    - name: Create a simple shell script to simulate the application
      copy:
        content: |
          #!/bin/bash
          echo "Starting test application server on port 5000..."
          cd /opt/{{ app_name }}
          python3 -m http.server 5000
        dest: "/opt/{{ app_name }}/run_test_app.sh"
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        mode: '0755'
        
    - name: Create a mock DLL file to satisfy the service
      copy:
        content: "This is not a real DLL file, just a placeholder"
        dest: "/opt/{{ app_name }}/{{ app_name }}.dll"
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        mode: '0755'
        
    - name: Create systemd service file directly
      copy:
        content: |
          [Unit]
          Description=Test ASP.NET Web App - {{ app_name }}
          After=network.target
          
          [Service]
          WorkingDirectory=/opt/{{ app_name }}
          ExecStart=/opt/{{ app_name }}/run_test_app.sh
          Restart=always
          RestartSec=10
          KillSignal=SIGINT
          SyslogIdentifier={{ app_name }}
          User={{ ansible_user }}
          EnvironmentFile=/etc/{{ app_name }}/.env
          
          [Install]
          WantedBy=multi-user.target
        dest: "/etc/systemd/system/{{ app_name }}.service"
        owner: root
        group: root
        mode: '0644'
      register: service_file
        
    - name: Reload systemd
      systemd:
        daemon_reload: yes
        
    - name: Start the test service
      systemd:
        name: "{{ app_name }}"
        state: started
        enabled: yes
EOF
```

### Step 7: Deploy and Test the Placeholder App

Run the playbook to deploy our test application:

```bash
ansible-playbook test_app_deploy.yml
```

Verify that the service is running:

```bash
ansible web -m command -a "systemctl status CloudSoft"
```

Try accessing the test application:

```bash
VM_IP=$(sed -n 's/.*ansible_host: \([0-9.]*\).*/\1/p' inventory.yaml)
curl http://${VM_IP}:5000
```

You should see the HTML from our test application.

### Step 8: Update Azure Network Security Group

Ensure your Azure VM allows traffic on port 5000:

```bash
# Open port 5000 on the Azure VM
az vm open-port \
  --resource-group AnsibleDemoRG \
  --name DemoVM \
  --port 5000 \
  --priority 1300
```

### Step 9: Clean Up the Test Application

Since this was just a test, let's stop the service until we're ready to deploy a real application:

```bash
cat > cleanup_test.yml << 'EOF'
---
- name: Stop Test Application
  hosts: web
  become: yes
  
  vars:
    app_name: CloudSoft
    
  tasks:
    - name: Stop the test service
      systemd:
        name: "{{ app_name }}"
        state: stopped
      ignore_errors: yes
EOF
```

Run the cleanup playbook:

```bash
ansible-playbook cleanup_test.yml -i inventory.yaml
```

## Troubleshooting

If you encounter issues during this exercise, here are some common solutions:

### Service Configuration Issues

- Check the systemd service file: `ansible web -i inventory.yaml -m command -a "cat /etc/systemd/system/CloudSoft.service"`
- Verify systemd is reloaded: `ansible web -i inventory.yaml -m command -a "systemctl daemon-reload"`
- Look at service logs: `ansible web -i inventory.yaml -m command -a "journalctl -u CloudSoft.service -n 50"`

### Environment File Issues

- Check the environment file: `ansible web -i inventory.yaml -m command -a "cat /etc/CloudSoft/.env"`
- Verify file permissions: `ansible web -i inventory.yaml -m command -a "ls -la /etc/CloudSoft/"`
- Test environment variables: `ansible web -i inventory.yaml -m command -a "grep ASPNETCORE_ENVIRONMENT /etc/CloudSoft/.env"`

### Test Application Issues

- Verify port 5000 is open: `ansible web -i inventory.yaml -m command -a "ss -tulpn | grep 5000"`
- Check process is running: `ansible web -i inventory.yaml -m command -a "ps aux | grep http.server"`
- Test local access: `ansible web -i inventory.yaml -m command -a "curl http://localhost:5000"`

## Final Tests

To verify that you've completed the exercise successfully:

1. Check that .NET is installed:

   ```bash
   ansible web -i inventory.yaml -m command -a "dotnet --info"
   ```

2. Verify the application directory structure:

   ```bash
   ansible web -i inventory.yaml -m command -a "ls -la /opt/CloudSoft"
   ```

3. Check the environment file:

   ```bash
   ansible web -i inventory.yaml -m command -a "cat /etc/CloudSoft/.env"
   ```

4. Verify the systemd service configuration:

   ```bash
   ansible web -i inventory.yaml -m command -a "systemctl cat CloudSoft"
   ```

5. Check service status (should be loaded but stopped after cleanup):

   ```bash
   ansible web -i inventory.yaml -m command -a "systemctl status CloudSoft"
   ```

✅ **Expected Results**

- .NET runtime is installed
- Application directories are created with proper permissions
- Environment file is in place with the correct settings
- Systemd service is configured correctly
- The service can be started and stopped
- The test application was accessible during testing

## Done! 🎉

Congratulations! You've successfully set up a .NET runtime environment on your server. You've created:

- A role for installing and configuring the .NET runtime
- A comprehensive application environment with proper directory structure
- Configuration for environment variables and application settings
- A systemd service for running .NET applications
- A test setup to verify your environment works

In the next exercise, we'll configure Nginx as a reverse proxy to securely expose your .NET application to the internet.
