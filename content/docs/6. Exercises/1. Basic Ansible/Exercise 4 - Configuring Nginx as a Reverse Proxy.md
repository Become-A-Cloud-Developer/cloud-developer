+++
title = "4. Configuring Nginx as a Reverse Proxy"
weight = 4
date = 2025-03-17
draft = false
+++

# Exercise 4: Configuring Nginx as a Reverse Proxy

## Goal

Configure Nginx as a secure reverse proxy for your .NET application, implementing SSL encryption and security best practices to safely expose your application to the internet.

## Learning Objectives

By the end of this exercise, you will:

- Generate and configure **SSL certificates** for secure communications
- Set up **Nginx as a reverse proxy** for your .NET application
- Implement **HTTP to HTTPS redirects** for improved security
- Configure **security headers** to protect against common web vulnerabilities
- Create **role dependencies** between nginx and proxy configuration
- Test and **validate the complete setup**
- Understand the **architecture of a web application deployment**

## Prerequisites

- Completed Exercises 1-3
- Access to your Azure VM
- Basic understanding of HTTP, HTTPS, and web security concepts

## Step-by-Step Instructions

### Step 1: Create a Secure Reverse Proxy Role

First, let's create a role for configuring Nginx as a reverse proxy with SSL:

1. Create the role directory structure:

    ```bash
    # Make sure you're in your ansible-learning directory
    cd ~/ansible-learning

    # Create the role directories
    mkdir -p roles/nginx_reverse_proxy/{tasks,handlers,defaults,templates,meta}
    ```

2. Create the main tasks file:

    ```bash
    cat > roles/nginx_reverse_proxy/tasks/main.yaml << 'EOF'
    ---
    - name: Create SSL directory
      file:
        path: /etc/nginx/ssl
        state: directory
        mode: '0755'

    - name: Generate SSL certificate
      shell: |
        openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
        -keyout /etc/nginx/ssl/nginx.key \
        -out /etc/nginx/ssl/nginx.crt \
        -subj "/C=US/ST=State/L=City/O=Organization/CN=localhost"
      args:
        creates: /etc/nginx/ssl/nginx.crt

    - name: Remove default nginx site
      file:
        path: "{{ item }}"
        state: absent
      with_items:
        - /etc/nginx/sites-enabled/default
        - /etc/nginx/sites-available/default
      notify: Restart nginx

    - name: Create reverse proxy configuration
      template:
        src: reverse-proxy.conf.j2
        dest: /etc/nginx/sites-available/reverse-proxy.conf
        mode: '0644'
      notify: Restart nginx

    - name: Enable site configuration
      file:
        src: /etc/nginx/sites-available/reverse-proxy.conf
        dest: /etc/nginx/sites-enabled/reverse-proxy.conf
        state: link
      notify: Restart nginx

    - name: Create security headers snippet
      template:
        src: security-headers.conf.j2
        dest: /etc/nginx/snippets/security-headers.conf
        owner: root
        group: root
        mode: '0644'
      notify: Reload nginx
    EOF
    ```

3. Create the handlers file:

    ```bash
    cat > roles/nginx_reverse_proxy/handlers/main.yaml << 'EOF'
    ---
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
        enabled: yes

    - name: Reload nginx
      service:
        name: nginx
        state: reloaded
    EOF
    ```

4. Create defaults file:

    ```bash
    cat > roles/nginx_reverse_proxy/defaults/main.yml << 'EOF'
    ---
    proxy_pass_port: 5000
    ssl_protocols: "TLSv1.2 TLSv1.3"
    ssl_prefer_server_ciphers: "on"
    ssl_ciphers: "ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384"
    EOF
    ```

5. Create the reverse proxy configuration template:

    ```bash
    cat > roles/nginx_reverse_proxy/templates/reverse-proxy.conf.j2 << 'EOF'
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
        
        ssl_protocols {{ ssl_protocols }};
        ssl_prefer_server_ciphers {{ ssl_prefer_server_ciphers }};
        ssl_ciphers {{ ssl_ciphers }};

        # Include security headers
        include snippets/security-headers.conf;

        location / {
            proxy_pass http://localhost:{{ proxy_pass_port }};
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
    EOF
    ```

6. Create the security headers template:

    ```bash
    cat > roles/nginx_reverse_proxy/templates/security-headers.conf.j2 << 'EOF'
    # Security headers
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-XSS-Protection "1; mode=block";
    add_header Referrer-Policy strict-origin-when-cross-origin;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self' data:; connect-src 'self'; frame-ancestors 'self'; form-action 'self';";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    EOF
    ```

7. Create role dependencies by creating a meta file:

    ```bash
    cat > roles/nginx_reverse_proxy/meta/main.yml << 'EOF'
    ---
    dependencies:
      - role: nginx
    EOF
    ```

> 💡 **Information**
>
> - The `shell` module generates a self-signed SSL certificate
> - `args: creates:` makes the shell command idempotent by checking if the file already exists
> - The configuration redirects all HTTP traffic to HTTPS
> - Security headers protect against common web vulnerabilities
> - `proxy_pass` forwards requests to your .NET application running on port 5000
> - Role dependencies ensure nginx is installed before configuring the reverse proxy

### Step 2: Update Azure Network Security Group

Make sure your Azure VM allows traffic on port 443 (HTTPS):

```bash
# Open port 443 on the Azure VM
az vm open-port \
  --resource-group AnsibleDemoRG \
  --name DemoVM \
  --port 443 \
  --priority 1400
```

### Step 3: Create a Playbook for Deploying the Reverse Proxy

Let's create a playbook to deploy our reverse proxy configuration:

```bash
cat > reverse_proxy_playbook.yml << 'EOF'
---
- name: Deploy Nginx Reverse Proxy
  hosts: web
  become: yes
  
  roles:
    - role: nginx_reverse_proxy
      vars:
        proxy_pass_port: 5000
EOF
```

### Step 4: Run the Playbook

Deploy the Nginx reverse proxy configuration:

```bash
ansible-playbook reverse_proxy_playbook.yml -i inventory.yaml
```

### Step 5: Redeploy the Test Application

Let's redeploy our test application from Exercise 3 to verify the reverse proxy:

```bash
cat > redeploy_test_app.yml << 'EOF'
---
- name: Redeploy Test Application
  hosts: web
  become: yes
  
  vars:
    app_name: CloudSoft
    
  tasks:
    - name: Create a simple test application file
      copy:
        content: >
          <!DOCTYPE html>
          <html>
          <head>
              <title>Test .NET Application</title>
              <style>
                  body {
                      font-family: Arial, sans-serif;
                      line-height: 1.6;
                      max-width: 800px;
                      margin: 0 auto;
                      padding: 20px;
                  }
                  .header {
                      background-color: #0078d4;
                      color: white;
                      padding: 20px;
                      border-radius: 5px;
                      margin-bottom: 20px;
                  }
                  .content {
                      background-color: #f5f5f5;
                      padding: 20px;
                      border-radius: 5px;
                  }
                  .footer {
                      margin-top: 20px;
                      text-align: center;
                      font-size: 0.8em;
                      color: #666;
                  }
              </style>
          </head>
          <body>
              <div class="header">
                  <h1>Test .NET Application</h1>
                  <p>Accessed through Nginx Reverse Proxy with SSL</p>
              </div>
              <div class="content">
                  <h2>Server Information</h2>
                  <ul>
                      <li>Hostname: {{ ansible_hostname }}</li>
                      <li>Environment: {{ lookup('env', 'ASPNETCORE_ENVIRONMENT') | default('Not Set', true) }}</li>
                      <li>Deployed at: {{ ansible_date_time.iso8601 }}</li>
                      <li>Connection: Secure HTTPS (Reverse Proxy)</li>
                  </ul>
                  <h2>Security Features</h2>
                  <ul>
                      <li>SSL/TLS Encryption</li>
                      <li>HTTP to HTTPS Redirection</li>
                      <li>Security Headers</li>
                      <li>Reverse Proxy Configuration</li>
                  </ul>
              </div>
              <div class="footer">
                  <p>Deployed using Ansible automation</p>
              </div>
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

Run the playbook to redeploy the test application:

```bash
ansible-playbook redeploy_test_app.yml -i inventory.yaml
```

### Step 6: Test the Reverse Proxy

Now let's test our reverse proxy setup:

1. Test HTTP to HTTPS redirection:

    ```bash
    VM_IP=$(sed -n 's/.*ansible_host: \([0-9.]*\).*/\1/p' inventory.yaml)
    curl -I http://${VM_IP}
    ```

    You should see a 301 redirect to HTTPS.

2. Test HTTPS access with SSL (ignoring certificate warnings since we're using a self-signed certificate):

    ```bash
    curl -k https://${VM_IP}
    ```

    You should see the HTML content of your test application.

3. Test security headers:

    ```bash
    curl -k -I https://${VM_IP} | grep -E 'Strict|Content|X-'
    ```

    You should see the security headers from our configuration.

### Step 7: Create a Complete Deployment Playbook

Let's create a comprehensive playbook that combines all the roles we've created:

```bash
cat > complete_deployment.yml << 'EOF'
---
- name: Complete Web Application Deployment
  hosts: web
  become: yes
  
  roles:
    # Install and configure Nginx for hello world site
    - role: nginx_hello_world
      vars:
        nginx_port: 8080
        site_title: "Hello from Ansible Automation!"
        site_heading: "Hello, Azure VM!"
        site_message: "This is a demo website deployed via Ansible roles"
        site_color: "#34A853"
      tags: [nginx, hello_world]
    
    # Setup .NET environment and application
    - role: dotnet_webapp
      vars:
        app_name: CloudSoft
        app_port: 5000
        app_environment: Production
        app_user: azureuser
        app_settings:
          CUSTOM_WELCOME_MESSAGE: "Hello from Azure VM!"
          DEPLOYMENT_ID: "{{ ansible_date_time.iso8601 }}"
      tags: [dotnet, application]
    
    # Configure Nginx as reverse proxy with SSL
    - role: nginx_reverse_proxy
      vars:
        proxy_pass_port: 5000
      tags: [nginx, reverse_proxy, ssl]
EOF
```

### Step 8: Run the Complete Playbook

Let's run our comprehensive deployment playbook:

```bash
ansible-playbook complete_deployment.yml -i inventory.yaml
```

This will set up all components of our web application environment.

### Step 9: Create Validation Tasks

Let's create a playbook to validate our complete setup:

```bash
cat > validate_setup.yml << 'EOF'
---
- name: Validate Complete Deployment
  hosts: web
  become: yes
  
  tasks:
    - name: Check Nginx status
      service:
        name: nginx
      register: nginx_status
      
    - name: Verify Nginx configuration
      command: nginx -t
      register: nginx_config
      changed_when: false
      
    - name: Check .NET application status
      service:
        name: CloudSoft
      register: app_status
      
    - name: Check SSL certificate
      command: openssl x509 -in /etc/nginx/ssl/nginx.crt -text -noout
      register: ssl_cert
      changed_when: false
      
    - name: Verify HTTP to HTTPS redirect
      uri:
        url: "http://localhost"
        method: HEAD
        status_code: 301
        follow_redirects: no
      register: redirect_check
      ignore_errors: yes
      
    - name: Check HTTPS access
      uri:
        url: "https://localhost"
        method: GET
        validate_certs: no
        return_content: yes
      register: https_check
      ignore_errors: yes
      
    - name: Check Hello World site
      uri:
        url: "http://localhost:8080"
        method: GET
        return_content: yes
      register: hello_world_check
      ignore_errors: yes
      
    - name: Display validation summary
      debug:
        msg: |
          Validation Summary:
          
          1. Nginx Status: {{ nginx_status.status.ActiveState | default('Unknown') }}
          2. Nginx Configuration: {{ 'Valid' if nginx_config.rc == 0 else 'Invalid' }}
          3. Application Status: {{ app_status.status.ActiveState | default('Unknown') }}
          4. SSL Certificate: {{ 'Present' if ssl_cert.rc == 0 else 'Missing' }}
          5. HTTP to HTTPS Redirect: {{ 'Working' if redirect_check.status == 301 else 'Not Working' }}
          6. HTTPS Access: {{ 'Working' if https_check.status == 200 else 'Not Working' }}
          7. Hello World Site: {{ 'Working' if hello_world_check.status == 200 else 'Not Working' }}
          
          Your environment is {{ 'completely set up' if 
            nginx_status.status.ActiveState == 'active' and 
            nginx_config.rc == 0 and 
            app_status.status.ActiveState == 'active' and 
            ssl_cert.rc == 0 and 
            redirect_check.status == 301 and 
            https_check.status == 200 and 
            hello_world_check.status == 200
            else 'partially set up - see details above' }}.
EOF
```

Run the validation playbook:

```bash
ansible-playbook validate_setup.yml -i inventory.yaml
```

## Troubleshooting

If you encounter issues during this exercise, here are some common solutions:

### SSL Certificate Issues

- Check if certificates were generated: `ansible web -i inventory.yaml -m command -a "ls -la /etc/nginx/ssl"`
- Verify openssl is installed: `ansible web -i inventory.yaml -m command -a "which openssl"`
- Check certificate details: `ansible web -i inventory.yaml -m command -a "openssl x509 -in /etc/nginx/ssl/nginx.crt -text -noout"`

### Nginx Configuration Issues

- Validate the configuration: `ansible web -i inventory.yaml -m command -a "nginx -t"`
- Check for syntax errors in templates
- Verify site is enabled: `ansible web -i inventory.yaml -m command -a "ls -la /etc/nginx/sites-enabled"`
- Look at error logs: `ansible web -i inventory.yaml -m command -a "tail /var/log/nginx/error.log"`

### Reverse Proxy Issues

- Check if app is running on port 5000: `ansible web -i inventory.yaml -m command -a "ss -tulpn | grep 5000"`
- Verify proxy pass configuration: `ansible web -i inventory.yaml -m command -a "grep -A 5 proxy_pass /etc/nginx/sites-enabled/reverse-proxy.conf"`
- Test locally on the server: `ansible web -i inventory.yaml -m command -a "curl -k https://localhost"`

### HTTPS Access Issues

- Verify port 443 is open in Azure NSG
- Check if Nginx is listening on port 443: `ansible web -i inventory.yaml -m command -a "ss -tulpn | grep 443"`
- Test redirection locally: `ansible web -i inventory.yaml -m command -a "curl -I http://localhost"`

## Final Tests

To verify that you've completed the exercise successfully:

1. Check that Nginx and the application are running:

   ```bash
   ansible web -i inventory.yaml -m command -a "systemctl status nginx"
   ansible web -i inventory.yaml -m command -a "systemctl status CloudSoft"
   ```

2. Verify the SSL configuration:

   ```bash
   ansible web -i inventory.yaml -m command -a "openssl x509 -in /etc/nginx/ssl/nginx.crt -noout -dates"
   ```

3. Test HTTP to HTTPS redirection:

   ```bash
   VM_IP=$(grep -oP "ansible_host: \K[0-9.]+" inventory.yaml)
   curl -I http://${VM_IP}
   ```

4. Test HTTPS access:

   ```bash
   curl -k https://${VM_IP}
   ```

5. Check the Hello World site:

   ```bash
   curl http://${VM_IP}:8080
   ```

6. Run the validation playbook again:

   ```bash
   ansible-playbook validate_setup.yml -i inventory.yaml
   ```

✅ **Expected Results**

- Nginx reverse proxy with SSL is configured correctly
- HTTP traffic is redirected to HTTPS
- Security headers are properly set
- The .NET application is accessible through the reverse proxy
- The Hello World site is accessible on port 8080
- All components are running as expected

## Architecture Overview

Your deployment now consists of the following components:

1. **Nginx Web Server**
   - Serves the Hello World site on port 8080
   - Acts as a reverse proxy with SSL for the .NET application

2. **.NET Runtime Environment**
   - Installed on the server
   - Configured with appropriate directories and permissions
   - Environment variables set via `.env` file

3. **Application Service**
   - Managed by systemd
   - Runs the .NET application on port 5000
   - Configured for automatic startup

4. **Secure Reverse Proxy**
   - Redirects HTTP to HTTPS
   - Terminates SSL/TLS connections
   - Forwards requests to the .NET application
   - Implements security headers

This architecture provides a secure, scalable foundation for hosting web applications, with clear separation of concerns between the different components.

## Done! 🎉

Congratulations! You've successfully completed all the exercises in the simplified Ansible learning roadmap. You've built a complete web application hosting environment with:

1. A configured Nginx web server
2. A .NET runtime environment
3. A secure reverse proxy with SSL

You've learned how to:

- Create and organize Ansible roles
- Use templates for dynamic configuration
- Work with handlers and notifications
- Manage services and configurations
- Implement security best practices
- Create a complete deployment pipeline

These skills provide a solid foundation for automating infrastructure and application deployments using Ansible. You can now expand on this knowledge by exploring more advanced topics like Ansible Vault for secrets management, dynamic inventories, or integrating with CI/CD pipelines.
