# Ansible Setup on AWS EC2

This guide explains how to install Ansible on an AWS EC2 instance and configure it to manage multiple Ubuntu EC2 instances using SSH authentication and an inventory file.

## Table of Contents

1. Introduction
2. Architecture
3. Prerequisites
4. Launch EC2 Instances
5. Install Ansible
6. Configure SSH Authentication
7. Configure the Inventory
8. Verify Connectivity
9. Troubleshooting
10. Security Best Practices

## 1. Introduction

Ansible is an open-source automation and configuration management tool used to automate server administration tasks such as installing packages, managing files, configuring services, and deploying applications.

Ansible is agentless by default. It connects to Linux servers using SSH and executes tasks remotely without requiring a separate Ansible agent on each managed server.

In this guide, Ansible is installed on one AWS EC2 instance, which acts as the control node and manages two additional EC2 instances.

## 2. Architecture

The setup consists of one control node and two managed nodes.

| Component | Description |
|---|---|
| Control Node | The EC2 instance where Ansible is installed and commands are executed. |
| Managed Nodes | The EC2 instances that Ansible configures and manages. |
| Inventory | A file containing the managed hosts and their connection details. |
| Modules | Units of work that perform tasks such as checking connectivity or installing packages. |
| Playbooks | YAML files that define tasks and the hosts on which those tasks should run. |

### Architecture Diagram

```text
                       AWS
                        |
               +------------------+
               |   Control Node   |
               |                  |
               |     Ansible      |
               |   inventory.ini  |
               +--------+---------+
                        |
                   SSH Connection
                    /          \
                   /            \
       +----------------+  +----------------+
       | Managed Node 1 |  | Managed Node 2 |
       |   Ubuntu EC2   |  |   Ubuntu EC2   |
       +----------------+  +----------------+
```

The control node communicates with both managed nodes over SSH to execute automation tasks.

## 3. Prerequisites

Before starting, ensure that you have:

- An AWS account.
- Three Ubuntu EC2 instances: one control node and two managed nodes.
- An SSH key pair for initially accessing the EC2 instances.
- SSH connectivity between the control node and managed nodes.
- Appropriate EC2 security group rules.

This guide uses Ubuntu Linux for the control node and managed nodes and the installation commands and SSH configuration are demonstrated using Ubuntu.

For this setup, Ubuntu is used on all three instances.

## 4. Launch EC2 Instances

### Step 1: Create the instances

Open the AWS Management Console and navigate to **EC2 → Instances → Launch instances**.

Create the following instances:

| Instance | Purpose |
|---|---|
| `ansible-control` | Runs Ansible and manages the other instances. |
| `ansible-managed-1` | First managed server. |
| `ansible-managed-2` | Second managed server. |

Select an Ubuntu AMI, an appropriate instance type, a key pair, and the required networking configuration.

### Step 2: Configure security groups

Configure the inbound rules to allow the required SSH connections.

| Instance | Protocol and Port | Source |
|---|---|---|
| Control Node | TCP 22 | Your trusted public IP address |
| Managed Nodes | TCP 22 | Control node's security group |

Using the control node's security group as the source for SSH access to the managed nodes is recommended when the instances are in the same VPC.

For communication over private IP addresses, ensure that the instances can reach one another within the VPC.

### Step 3: Connect to the control node

Connect to the control node using SSH:

```bash
ssh -i /path/to/aws-key.pem ubuntu@<CONTROL_NODE_PUBLIC_IP>
```

Replace the key path and IP address with your actual values.

The following commands are executed on the control node unless specified otherwise.

## 5. Install Ansible

### Step 1: Update the package lists

```bash
sudo apt update
```

### Step 2: Install the required package

```bash
sudo apt install -y software-properties-common
```

### Step 3: Add the Ansible repository

```bash
sudo apt-add-repository --yes --update ppa:ansible/ansible
```

### Step 4: Install Ansible

```bash
sudo apt install -y ansible
```

### Step 5: Verify the installation

```bash
ansible --version
```

If the installation is successful, the command displays the installed Ansible version and configuration information.

## 6. Configure SSH Authentication

Ansible uses SSH to connect to managed nodes. To enable these connections, configure a dedicated SSH key on the control node and add its public key to both managed nodes.

### Step 1: Generate an SSH key pair

On the control node, run:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519
```

Follow the prompts to configure the key. A passphrase can be added for additional security.

The command generates two files:

- `~/.ssh/id_ed25519` — the private key.
- `~/.ssh/id_ed25519.pub` — the public key.

Protect the private key:

```bash
chmod 600 ~/.ssh/id_ed25519
```

Display the public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

### Step 2: Add the public key to the managed nodes

The public key generated on the control node must be added to the Ubuntu user's `~/.ssh/authorized_keys` file on both managed nodes.

Use your existing AWS EC2 key pair or another authorized access method to establish the initial connection. Then append the control node's public key to `authorized_keys` on each managed node.

For example, after connecting to a managed node as `ubuntu`, run:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

Add the complete public key from the control node on a new line. Save the file, then run:

```bash
chmod 600 ~/.ssh/authorized_keys
```

Repeat these steps on the second managed node.

**Important:** Add only the public key to the managed nodes. Never copy or expose the private key.

### Step 3: Test SSH connectivity

From the control node, connect to each managed node:

```bash
ssh -i ~/.ssh/id_ed25519 ubuntu@<MANAGED_NODE_1_IP>
```

```bash
ssh -i ~/.ssh/id_ed25519 ubuntu@<MANAGED_NODE_2_IP>
```

Replace the placeholders with the actual IP addresses.

If the connections succeed, SSH authentication is configured. Exit each remote session before continuing:

```bash
exit
```

## 7. Configure the Inventory

An inventory file defines the servers that Ansible manages and groups them for easier administration.

### Step 1: Create the inventory file

On the control node, run:

```bash
mkdir -p ~/ansible-setup
cd ~/ansible-setup
touch inventory.ini
```

Open the file:

```bash
nano inventory.ini
```

### Step 2: Add the configuration

Add the following content:

```ini
[servers]
server1 ansible_host=<MANAGED_NODE_1_IP>
server2 ansible_host=<MANAGED_NODE_2_IP>

[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_user=ubuntu
ansible_ssh_private_key_file=/home/ubuntu/.ssh/id_ed25519
```

Replace the IP address placeholders with the actual IP addresses of your managed instances.

If your control node uses a different Linux username or private key location, update the relevant values.

### Step 3: Understand the configuration

**`[servers]`**

Defines a group named `servers` containing the managed nodes.

**`server1` and `server2`**

These are inventory aliases used to identify the managed nodes in Ansible commands.

**`ansible_host`**

Specifies the actual IP address or hostname of each managed node.

**`ansible_user`**

Specifies the SSH username used to connect to the managed nodes. For Ubuntu EC2 instances, this is commonly `ubuntu`.

**`ansible_ssh_private_key_file`**

Specifies the private key Ansible uses for SSH authentication.

**`ansible_python_interpreter`**

Specifies the Python interpreter used by Ansible modules that require Python on the managed nodes.

**`[all:vars]`**

Defines variables that apply to all hosts in the inventory unless overridden by more specific host or group settings.

### Step 4: Verify the inventory

Run:

```bash
ansible -i inventory.ini servers --list-hosts
```

The output should list both managed nodes in the `servers` group.

## 8. Verify Connectivity

After configuring SSH authentication and the inventory, verify that Ansible can communicate with the managed nodes.

Run the following command from the directory containing `inventory.ini`:

```bash
ansible -i inventory.ini servers -m ping
```

### Expected Output

```text
server1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
server2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

The exact output may differ depending on the installed Ansible version.

### Command Explanation

- `-i inventory.ini` specifies the inventory file.
- `servers` selects the hosts in the `servers` group.
- `-m ping` executes Ansible's built-in ping module on the selected hosts.
- `SUCCESS` with `"ping": "pong"` indicates that the module executed successfully.

Ansible's ping module is not an ICMP network ping. It checks whether Ansible can connect to the remote host and execute the module successfully.

If both managed nodes return `SUCCESS`, the initial setup is complete.

## 9. Troubleshooting

| Issue | Possible Solution |
|---|---|
| `UNREACHABLE` | Verify IP addresses, network connectivity, SSH access, and security group rules. |
| `Permission denied (publickey)` | Check the SSH username, private key path, and public key in `authorized_keys`. |
| Private key permission error | Run `chmod 600 ~/.ssh/id_ed25519` and verify file ownership. |
| Host key verification failure | Verify the server's identity and inspect the relevant entry in `~/.ssh/known_hosts`. |
| Python interpreter error | Confirm Python 3 is installed on the managed node and the configured interpreter path is correct. |
| SSH connection timeout | Verify that the control node can reach port 22 on the managed node. |

For detailed connection diagnostics, run:

```bash
ansible -i inventory.ini servers -m ping -vvv
```

Review the output to identify the source of the connection failure.

## 10. Security Best Practices

- Never commit private SSH keys, AWS credentials, or secrets to GitHub.
- Restrict SSH access to trusted IP addresses and security groups.
- Prefer private IP addresses for communication between instances in the same VPC.
- Keep private key permissions restrictive.
- Verify SSH host keys instead of disabling host key checking.
- Use IAM roles and other appropriate AWS security controls where applicable.
- Stop or terminate lab instances when they are no longer required to avoid unnecessary AWS charges.

## Conclusion

In this guide, Ansible was installed on an AWS EC2 control node and configured to manage two Ubuntu EC2 instances.

The setup covered:

- Installing Ansible on the control node.
- Configuring SSH authentication.
- Creating an inventory file to organize managed nodes.
- Verifying connectivity using Ansible's ping module.

This setup provides the foundation for automating server administration tasks, including package installation, service management, file operations, and application deployment using Ansible.

## References

- [Ansible Documentation](https://docs.ansible.com/)
- [Ansible Installation Guide](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
- [Ansible Inventory Guide](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html)
- [AWS EC2 User Guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html)
- [Terraform Guide Repository](https://github.com/vaishnavipawardottech/Terraform-Guide)