# Ansible Overview & Basic Usage
### What is Ansible?
Ansible is a configuration management tool used to manage both software and hardware across multiple servers. It allows you to automate tasks like software installation, updates, and system configurations.

One key advantage of Ansible is that it is agentless, meaning only the control node requires Ansible installation, while the worker nodes (managed servers) do not need any additional software. Ansible runs on Python and uses SSH to communicate with worker nodes.


## Setting Up SSH Connection (Control Node to Worker Node)
To manage worker nodes, Ansible connects via SSH. You can authenticate using either password-based login or passwordless authentication (recommended).
### Steps for Password-Based SSH Authentication:
1. On the worker node, set a root password using:
   ```
   passwd
   ```

2. Modify SSH configurations:
   - Open /etc/ssh/sshd_config.d/60-custom.conf (for newer systems) or /etc/ssh/sshd_config (for older systems).
   - Set the following values:
   ```
   PermitRootLogin yes
   PasswordAuthentication yes
   ```
   - Save and exit.

3. Restart the SSH service:
   - For RHEL-based systems (RedHat, CentOS, Rocky Linux, etc):
     ```
     systemctl restart sshd
     ```

   - For Debian-based systems (Ubuntu, Debian, etc.):
     ```
      systemctl restart ssh
     ```

## Setting Up Passwordless SSH Authentication (Recommended)

1. On the control node, generate an SSH key pair:

```
ssh-keygen
```

2. Copy the public key to the worker node:
```
ssh-copy-id -f "-o identityfile <pathtokeypair>" ubuntu@<ip-address>
```

Enter the root password when prompted. Once done, the control node can connect without requiring a password.

3. To confirm connectivity, run:

```
ssh root@<worker-node-ip>
```
If successful, it should log in without asking for a password.
     
   
## Testing Ansible Connection
Once SSH is set up, test the connection using Ansible:

```
ansible all -i inventory.ini -m ping
```

A successful connection will return:

```
<worker-ip> | SUCCESS => {...}
```

## Ansible Ad-Hoc Commands vs Playbooks
- Ad-Hoc Commands → Used for simple, one-time tasks. Example:

```
ansible all -i inventory.ini -m apt -a "name=apache2 state=present" --become
```

- Playbooks → Used for automating multiple tasks in a structured manner. Example:

```
ansible-playbook -i inventory.ini playbook.yml
```

## Gathering System Information
To check system details of worker nodes, use:

```
ansible all -i inventory.ini -m setup
```

This fetches detailed facts about each server, including OS version, CPU, memory, and more.

## Next Steps: Ansible Roles & Vaults
Ansible Roles → Organizing playbooks for better reusability and structure.

Ansible Vault → Encrypting sensitive data (e.g., passwords, API keys) in playbooks.

## Understanding Ansible Roles
Ansible roles improve readability and modularity, making it easier to manage complex playbooks. Instead of writing long playbooks with many tasks, roles allow us to organize tasks into structured directories.

For example, if we need to install Docker on multiple OS-managed nodes, the steps will vary for Ubuntu, CentOS, and RedHat. Instead of writing all these conditions in a single playbook, we create roles to handle each scenario separately.

### Ways to Use Ansible Roles
There are two main ways to use Ansible roles:

1. Creating Your Own Roles
2. Using Prebuilt Roles from Ansible Galaxy

### Creating Your Own Ansible Role
To create a new role, run:

```
ansible-galaxy role init <role_name>
```

This will generate a folder structure like this:

```
install_docker/
│── defaults/        # Stores default variables
│── files/           # Stores files to copy (e.g., config files)
│── handlers/        # Defines handlers for task dependencies
│── meta/            # Metadata about the role
│── tasks/           # Defines tasks (main.yaml)
│── templates/       # Stores Jinja2 template files
│── vars/            # Stores role-specific variables
│── README.md        # Documentation for the role
```

### Explanation of Role Structure:
- tasks/ → The main directory where we define tasks (like installing Docker).
- files/ → Stores files we want to copy (e.g., index.html, docker-config.json).
- vars/ vs defaults/ → Both store variables, but defaults have lower priority, meaning values in vars/ override those in defaults/.
- handlers/ → Defines actions that should run after specific tasks (e.g., restarting a service after installation).

### Example: Installing Docker Using Ansible Roles

Step 1: Define Tasks (Inside tasks/main.yml)

```
---
- name: Install Docker (Ubuntu)
  apt:
    name: docker.io
    state: present
  when: ansible_os_family == "Debian"

- name: Install Docker (CentOS)
  yum:
    name: docker
    state: present
  when: ansible_os_family == "RedHat"

- name: Start and Enable Docker
  service:
    name: docker
    state: started
    enabled: true
```


Step 2: Use Role in a Playbook (playbook.yml)

```
---
- hosts: all
  become: true
  roles:
    - install_docker

```

Step 3: Run the Playbook

```
ansible-playbook -i inventory.ini playbook.yml
```


### Using Ansible Galaxy
Ansible Galaxy is a repository of prebuilt roles that we can download instead of creating roles from scratch.

To install a role from Galaxy in galaxy.ansible.com, where you can also push your own roles or copy existing roles from others run:

```
ansible-galaxy install <role_name>
```

### Locate the Installed Role

Once installed, the role is stored in:

```
~/.ansible/roles/<role_name>

```

### Use the Role in a Playbook
Now, reference the installed role inside your playbook.yml:

```
---
- hosts: all
  become: true
  roles:
    - geerlingguy.docker
```


## Ansible Collections
Ansible Collections allow us to connect to third-party services (e.g., AWS, Azure, Cisco) using APIs.

For example, to manage AWS resources, we need Boto3, which is an AWS SDK for Python.

Step 1: Install Boto3

```
pip install boto3

```

Step 2: Create a Collection

```
ansible-galaxy collection install amazon.aws
```

This generates a folder structure similar to roles but designed for third-party integrations.

## Ansible error handling
By default, Ansible stops execution when it encounters an error. However, we can handle errors using ignore_errors: yes to allow the playbook to continue executing even if a task fails.

### Example: Checking & Handling Errors for OpenSSH, OpenSSL, and Docker Installation
Let's say we need to ensure that OpenSSH and OpenSSL are updated on all worker servers, then check if Docker is installed.

Without Error Handling (Execution Stops on Failure)

```
- hosts: all
  become: true
  tasks:
    - name: Checking OpenSSH and OpenSSL
      ansible.builtin.apt:
        name: "{{ item }}"
        state: latest
      loop:
        - openssh
        - openssl

    - name: Checking Docker Version
      command: docker --version
```

- If OpenSSH or OpenSSL is already updated, the task will complete successfully.
- However, if Docker is not installed, the second task will fail, and Ansible will stop execution.

### With Error Handling (Execution Continues Even on Failure)

```
- hosts: all
  become: true
  tasks:
    - name: Checking OpenSSH and OpenSSL
      ansible.builtin.apt:
        name: "{{ item }}"
        state: latest
      loop:
        - openssh
        - openssl
      ignore_errors: yes  # Ignore errors and continue execution

    - name: Checking Docker Version
      command: docker --version
      register: output
      ignore_errors: yes  # Ignore error if Docker is not installed

    - name: Debug Docker Output
      ansible.builtin.debug:
        var: output

    - name: Install Docker if Not Present
      apt:
        name: docker.io
        state: present
      when: output.failed is defined and output.rc != 0  # Install Docker only if it is missing
```

### Explanation:
- The first task ensures OpenSSH and OpenSSL are updated.
- If an error occurs, ignore_errors: yes ensures the playbook continues execution instead of stopping.
- The second task checks if Docker is installed.
- If Docker is not installed, the error is stored in output.
- The third task prints the output to debug the error.
- The final task installs Docker only if an error was detected (Docker is missing).

This approach ensures a smooth playbook execution without unnecessary failures stopping the workflow.


   
