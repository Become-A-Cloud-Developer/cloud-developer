+++
title = "1. Environment Setup & Ansible Installation"
weight = 1
date = 2025-03-17
draft = false
+++

# Exercise 1: Environment Setup & Ansible Installation

## Goal

Set up a consistent and isolated Ansible environment using Python virtual environments across different operating systems, ensuring you have a proper foundation for all subsequent exercises.

## Learning Objectives

By the end of this exercise, you will:

- Create **Python virtual environments** to isolate Ansible installations
- Install **Ansible** via pip in a controlled environment
- Understand **Ansible command-line basics** and version information
- Create a simple **inventory file** for Ansible
- Execute your first **ad-hoc command** against localhost
- Configure **Ansible settings** through ansible.cfg

## Prerequisites

- Basic command-line familiarity (Terminal on macOS/Linux, Command Prompt or PowerShell on Windows)
- Python 3.8 or newer installed on your system
- A text editor of your choice (VS Code recommended)

## Step-by-Step Instructions

### Step 1: Verify Python Installation

1. Open a terminal or command prompt on your system.
2. Check that Python is installed and is version 3.8 or newer:

  ```bash
  python --version
  # or
  python3 --version
  ```

> 💡 **Information**
>
> - If Python is not installed or is older than 3.8, download the latest version from [python.org](https://www.python.org/downloads/).
> - On macOS/Linux, you might need to use `python3` instead of `python`.
> - On Windows, ensure Python is added to your PATH during installation.

### Step 2: Create a Project Directory

1. Create a directory for your Ansible learning project:

  ```bash
  # macOS/Linux
  mkdir -p ~/ansible-learning
  cd ~/ansible-learning

  # Windows (Command Prompt)
  mkdir %USERPROFILE%\ansible-learning
  cd %USERPROFILE%\ansible-learning

  # Windows (PowerShell)
  mkdir $HOME\ansible-learning
  cd $HOME\ansible-learning
  ```

### Step 3: Set Up a Python Virtual Environment

#### On macOS/Linux

1. Create a virtual environment:

    ```bash
    python3 -m venv venv
    ```

2. Activate the virtual environment:

    ```bash
    source venv/bin/activate
    ```

> 💡 **Information**
>
> - Your command prompt should now show `(venv)` at the beginning, indicating the virtual environment is active.
> - To deactivate the environment later, simply type `deactivate`.

#### On Windows (Command Prompt)

1. Create a virtual environment:

    ```cmd
    python -m venv venv
    ```

2. Activate the virtual environment:

    ```cmd
    venv\Scripts\activate
    ```

#### On Windows (PowerShell)

1. Create a virtual environment:

    ```powershell
    python -m venv venv
    ```

2. Activate the virtual environment:

    ```powershell
    .\venv\Scripts\Activate.ps1
    ```

> 💡 **Information**
>
> - If you encounter an execution policy error in PowerShell, you might need to change the execution policy temporarily:
>
>   ```powershell
>   Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope Process
>   ```
>
> - This only affects the current PowerShell session.

### Step 4: Install Ansible

With your virtual environment activated, install Ansible using pip:

```bash
pip install ansible
```

> 💡 **Information**
>
> - Using pip in a virtual environment installs Ansible in an isolated way, avoiding conflicts with system packages.
> - This allows you to have multiple versions of Ansible for different projects on the same machine.
> - Pip automatically installs all dependencies required by Ansible.

### Step 5: Verify Ansible Installation

Check that Ansible is installed correctly:

```bash
ansible --version
```

You should see output similar to:

```terminal
ansible [core 2.14.x]
  config file = None
  configured module search path = [...]
  ansible python module location = /path/to/venv/lib/python3.x/site-packages/ansible
  ansible collection location = /path/to/home/.ansible/collections:/usr/share/ansible/collections
  executable location = /path/to/venv/bin/ansible
  python version = 3.x.x (...)
  jinja version = 3.x.x
  libyaml = True
```

### Step 6: Create an Ansible Configuration File

1. Create a basic `ansible.cfg` file in your project directory:

    ```bash
    # macOS/Linux
    touch ansible.cfg

    # Windows (Command Prompt)
    type nul > ansible.cfg

    # Windows (PowerShell)
    New-Item -Path "ansible.cfg" -ItemType "file"
    ```

2. Open the file in your text editor and add:

    ```ini
    [defaults]
    inventory = inventory.yaml
    host_key_checking = False
    ```

> 💡 **Information**
>
> - `inventory = inventory.yaml` tells Ansible where to find your hosts file.
> - `host_key_checking = False` disables SSH host key checking, which is helpful for learning environments (but not recommended for production).
> - Ansible uses a hierarchy of configuration files, with this local file having the highest precedence.

### Step 7: Create Your First Inventory File

1. Create an inventory file named `inventory.yaml`:

    ```bash
    # macOS/Linux
    touch inventory.yaml

    # Windows (Command Prompt)
    type nul > inventory.yaml

    # Windows (PowerShell)
    New-Item -Path "inventory.yaml" -ItemType "file"
    ```

2. Open the file in your text editor and add:

    ```yaml
    all:
      hosts:
        localhost:
          ansible_connection: local
    ```

> 💡 **Information**
>
> - This minimal inventory file defines a single host named `localhost`.
> - `ansible_connection: local` tells Ansible to execute commands directly on the local machine without using SSH.
> - Ansible inventories can be in YAML format (like above) or INI format.

### Step 8: Run Your First Ansible Command

Now let's run a simple ad-hoc command to verify everything works:

```bash
ansible localhost -m ping
```

You should see output similar to:

```terminal
localhost | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

> 💡 **Information**
>
> - The `ping` module verifies that Ansible can communicate with the host.
> - `-m ping` specifies the module to use.
> - This command doesn't perform a network ping - it verifies that Ansible can run Python on the target host.

### Step 9: Explore Ansible Modules and Documentation

Let's run one more command to see information about your system:

```bash
ansible localhost -m setup
```

> 💡 **Information**
>
> - The `setup` module gathers "facts" about the target system.
> - The output includes details about your operating system, hardware, network configuration, and more.
> - These facts can be used in Ansible playbooks to make decisions based on host characteristics.

## Troubleshooting

If you encounter issues during this exercise, here are some common solutions:

### Python Not Found or Incorrect Version

- Ensure Python 3.8+ is installed and in your PATH.
- On macOS/Linux, try using `python3` instead of `python`.

### Virtual Environment Creation Fails

- Ensure you have the `venv` module installed. On some systems, it might need to be installed separately:

  ```bash
  # On Ubuntu/Debian
  sudo apt-get install python3-venv
  ```

### Activation Script Not Found

- Verify that the virtual environment was created successfully by checking if the `venv` directory exists.
- On Windows, ensure you're using the correct path separator (`\` for Command Prompt/PowerShell).

### Ansible Not Found After Installation

- Make sure your virtual environment is activated (you should see `(venv)` in your prompt).
- Try reinstalling Ansible: `pip install --force-reinstall ansible`.

### Permission Errors on Windows

- If you encounter PowerShell execution policy errors, run:

  ```powershell
  Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope Process
  ```

## Final Tests

To verify that you've completed the exercise successfully, check the following:

1. Confirm that your virtual environment is activated (you should see `(venv)` in your prompt).

2. Run this command to check Ansible's version:

   ```bash
   ansible --version
   ```

   The output should show the Ansible version and configuration details.

3. Run the ping module against localhost:

   ```bash
   ansible localhost -m ping
   ```

   You should see a successful "pong" response.

4. Verify your project directory has these files:

   - `ansible.cfg`
   - `inventory.yaml`

✅ **Expected Results**

- You have a working Python virtual environment with Ansible installed.
- You can run ad-hoc Ansible commands against localhost.
- You understand the basic structure of an inventory file and configuration.
- You're ready to start writing and running Ansible playbooks in the next exercise!

## Done! 🎉

Congratulations! You've successfully set up your Ansible environment. You now have a solid foundation for the rest of the exercises in this series. In the next exercise, we'll dive into writing your first Ansible playbook!
