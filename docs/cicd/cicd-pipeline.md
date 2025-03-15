# Setting Up CI/CD for Your Homelab

This guide covers how to set up a secure CI/CD pipeline for deploying applications to your homelab environment using GitHub Actions (or other CI/CD platforms).

## Overview

A proper CI/CD setup allows you to:
- Automatically deploy code changes to your homelab
- Maintain separation between development and production environments
- Follow security best practices for automated deployments
- Manage persistent storage for your applications

## Prerequisites

- A homelab server running Linux
- SSH access to your server
- GitHub repository (or other CI/CD platform)
- NAS or other persistent storage solution

## Step 1: Create a Dedicated CI/CD User

SSH into your homelab server and create a dedicated user for CI/CD operations:

```bash
# Create the user
sudo useradd -m -s /bin/bash cicd-bot

# Create .ssh directory with proper permissions
sudo mkdir -p /home/cicd-bot/.ssh
sudo chmod 700 /home/cicd-bot/.ssh

# Create authorized_keys file
sudo touch /home/cicd-bot/.ssh/authorized_keys
sudo chmod 600 /home/cicd-bot/.ssh/authorized_keys
sudo chown -R cicd-bot:cicd-bot /home/cicd-bot/.ssh
```

## Step 2: Set Up Docker Permissions

The CI/CD user needs permission to run Docker commands:

```bash
# Add cicd-bot to the docker group
sudo usermod -aG docker cicd-bot
```

## Step 3: Generate an SSH Key Pair for CI/CD

On your local machine (not the server):

```bash
# Generate a key pair specifically for CI/CD
ssh-keygen -t ed25519 -C "cicd-bot@github-actions" -f cicd-bot_github_actions

# This creates:
# - cicd-bot_github_actions (private key)
# - cicd-bot_github_actions.pub (public key)
```

## Step 4: Add the Public Key to the Server

Copy the contents of the public key and add it to the authorized_keys file on your server:

```bash
# On your local machine, display the public key
cat cicd-bot_github_actions.pub

# Copy the output, then on your server:
sudo bash -c 'echo "ssh-ed25519 AAAA...your key here... cicd-bot@github-actions" >> /home/cicd-bot/.ssh/authorized_keys'
```

## Step 5: Get the SSH Known Hosts Entry

To get the known hosts entry for your CI/CD platform secret:

```bash
# Replace tailscale-ip with your server's Tailscale IP address
ssh-keyscan tailscale-ip
```

Note: You should use the Tailscale IP address of your server for secure connections through the Tailscale network.

## Step 6: Set Up CI/CD Secrets

Add these secrets to your GitHub repository (or equivalent in your CI/CD platform):

- `HOMELAB_HOST`: Your server's Tailscale IP address
- `HOMELAB_USER`: Set to `cicd-bot`
- `HOMELAB_SSH_KEY`: The entire contents of the private key file (cicd-bot_github_actions)
- `HOMELAB_SSH_KNOWN_HOSTS`: The output from the ssh-keyscan command

## Step 7: Test the Connection

Test the connection from your local machine:

```bash
ssh -i cicd-bot_github_actions cicd-bot@your-homelab-host
```

## Step 8: Set Up Deployment Directories

You have two options for setting up the deployment directories:

### Option 1: Manual Setup on the Server

```bash
# Create a deployment directory
sudo mkdir -p /opt/deployments

# Give cicd-bot access to the directory
sudo chown -R cicd-bot:cicd-bot /opt/deployments
```

### Option 2: Group-Based Access (For Multiple Users)

If multiple users need access:

```bash
# Create a group
sudo groupadd deploy-group

# Add cicd-bot to the group
sudo usermod -aG deploy-group cicd-bot

# Set group ownership and permissions
sudo mkdir -p /opt/deployments
sudo chown -R root:deploy-group /opt/deployments
sudo chmod -R 770 /opt/deployments
```

### Option 3: Automated Setup via CI/CD Pipeline

A more automated approach is to have your CI/CD pipeline create the necessary directories. This ensures the directories exist and have the correct permissions before deployment:

```yaml
- name: Create Directory Structure
  run: |
    ssh ${{ secrets.HOMELAB_USER }}@${{ secrets.HOMELAB_HOST }} << 'ENDSSH'
      # Create deployment directory structure
      sudo mkdir -p /opt/deployments/{prod,staging}/{app,backups}
      
      # Create docker volumes directory structure
      sudo mkdir -p /mnt/nas-share/docker/volumes/{service1,service2}/{prod,staging}
      
      # Set ownership
      sudo chown -R ${{ secrets.HOMELAB_USER }}:${{ secrets.HOMELAB_USER }} /opt/deployments
      sudo chown -R ${{ secrets.HOMELAB_USER }}:${{ secrets.HOMELAB_USER }} /mnt/nas-share/docker/volumes/{service1,service2}
      
      # Set permissions
      sudo chmod -R 755 /opt/deployments
      sudo chmod -R 755 /mnt/nas-share/docker/volumes/{service1,service2}
      
      echo "✅ Directory structure created successfully"
    ENDSSH
```

This approach has several advantages:
- Ensures directories exist before deployment
- Creates a consistent directory structure
- Sets appropriate permissions automatically
- Can be version-controlled with your application code
- Works well for multi-environment setups (prod, staging, dev)

## Step 9: Set Up Persistent Storage

Create directories on your NAS or other storage for different environments:

```bash
# Create directories for different services and environments
mkdir -p /mnt/nas-share/docker/volumes/{service1,service2,service3}/{prod,staging}
```

This creates a structure like:
- `/mnt/nas-share/docker/volumes/service1/prod`
- `/mnt/nas-share/docker/volumes/service1/staging`
- `/mnt/nas-share/docker/volumes/service2/prod`
- etc.

## Step 10: Restrict the CI/CD User (Recommended)

For added security, restrict what the cicd-bot user can do:

```bash
# Create a sudoers file for cicd-bot with limited permissions
sudo bash -c 'cat > /etc/sudoers.d/cicd-bot << EOF
# Allow cicd-bot to use Docker without password
cicd-bot ALL=(ALL) NOPASSWD: /usr/bin/docker
EOF'

sudo chmod 440 /etc/sudoers.d/cicd-bot
```

## Step 11: Create a GitHub Actions Workflow

Create a `.github/workflows/deploy.yml` file in your repository:

```yaml
name: Deploy to Homelab

on:
  push:
    branches: [ main ]  # Or your production branch
  workflow_dispatch:    # Allow manual triggers

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Tailscale
        uses: tailscale/github-action@v3
        with:
          authkey: ${{ secrets.TAILSCALE_AUTHKEY }}
          tags: tag:ci
          version: latest
      
      - name: Configure SSH
        uses: shimataro/ssh-key-action@v2
        with:
          key: ${{ secrets.SSH_KEY }}
          known_hosts: ${{ secrets.SSH_KNOWN_HOSTS }}
      
      - name: Create Directory Structure
        run: |
          ssh ${{ secrets.HOMELAB_USER }}@${{ secrets.HOMELAB_HOST }} << 'ENDSSH'
            # Create deployment directory structure
            sudo mkdir -p /opt/deployments/{prod,staging}/{app,backups}
            
            # Create docker volumes directory structure if using NAS storage
            sudo mkdir -p /mnt/nas-share/docker/volumes/{service1,service2}/{prod,staging}
            
            # Set ownership
            sudo chown -R ${{ secrets.HOMELAB_USER }}:${{ secrets.HOMELAB_USER }} /opt/deployments
            sudo chown -R ${{ secrets.HOMELAB_USER }}:${{ secrets.HOMELAB_USER }} /mnt/nas-share/docker/volumes/{service1,service2}
            
            # Set permissions
            sudo chmod -R 755 /opt/deployments
            sudo chmod -R 755 /mnt/nas-share/docker/volumes/{service1,service2}
            
            echo "✅ Directory structure created successfully"
          ENDSSH
      
      - name: Sync files to server
        run: |
          rsync -avz --delete \
            --exclude '.git/' \
            --exclude '.github/' \
            --exclude 'node_modules/' \
            --exclude '.gitignore' \
            ./ ${{ secrets.HOMELAB_USER }}@${{ secrets.HOMELAB_HOST }}:/opt/deployments/
      
      - name: Deploy to homelab
        run: |
          ssh ${{ secrets.HOMELAB_USER }}@${{ secrets.HOMELAB_HOST }} << 'EOF'
            cd /opt/deployments
            
            # Deploy using Docker Compose
            docker compose -f docker-compose.prod.yml up -d --build
          EOF
```

This workflow:
1. Checks out your code
2. Sets up Tailscale using a GitHub Action to create a secure connection
3. Configures SSH with your private key and known hosts
4. Creates the necessary directory structure on the server
5. Syncs your files to the server using rsync (excluding unnecessary files)
6. Connects to your server via SSH through the Tailscale network
7. Executes deployment commands on your server

### Customizing the Rsync Command

The rsync command above:
- Uses `-avz` for archive mode, verbosity, and compression
- Uses `--delete` to remove files on the server that are no longer in the repository
- Excludes common directories and files that shouldn't be deployed:
  - `.git/`: Git repository data
  - `.github/`: GitHub-specific files like workflows
  - `node_modules/`: Dependencies that should be installed on the server
  - `.gitignore`: Git configuration file

You should customize the excluded paths based on your project structure and requirements.

### Required Secrets

For this workflow, you'll need to set up these GitHub secrets:

- `TAILSCALE_AUTHKEY`: An auth key generated from your Tailscale admin console
- `SSH_KEY`: Your private SSH key (same as HOMELAB_SSH_KEY mentioned earlier)
- `SSH_KNOWN_HOSTS`: The output from the ssh-keyscan command
- `HOMELAB_USER`: Your CI/CD user (cicd-bot)
- `HOMELAB_HOST`: Your server's Tailscale IP address

### Tailscale Auth Key

To generate a Tailscale auth key:
1. Go to the [Tailscale Admin Console](https://login.tailscale.com/admin/settings/keys)
2. Click "Generate auth key"
3. Set appropriate key expiry and permissions
4. Use the "tag:ci" option to identify connections from your CI/CD pipeline

## Security Considerations

- The CI/CD user should have the minimum permissions needed
- Use separate environments for staging and production
- Consider using Docker secrets for sensitive information
- Regularly rotate SSH keys
- Monitor CI/CD logs for unauthorized access attempts

## Troubleshooting

### SSH Connection Issues

If you can't connect via SSH:
```bash
# Check SSH service status
sudo systemctl status sshd

# Check SSH logs
sudo journalctl -u sshd

# Verify permissions on .ssh directory
ls -la /home/cicd-bot/.ssh
```

### Docker Permission Issues

If the CI/CD user can't run Docker commands:
```bash
# Verify the user is in the docker group
groups cicd-bot

# Restart the Docker service
sudo systemctl restart docker
```

## Next Steps

- [Set up monitoring for your CI/CD pipeline](monitoring.md)
- [Implement automated testing](testing.md)
- [Configure backup solutions](../backup/README.md) 