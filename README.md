# Laravel in Shared Hosting (Go54) Deployment Guide

This document provides a step-by-step guide for deploying a Laravel application to a Go54 (or similar cPanel-based) shared hosting environment using GitHub Actions for Continuous Deployment (CD).

We will use FTP for file transfer and then transition to SSH for automated post-deployment tasks like running migrations and clearing caches.

## Phase 1: Initial Server and Environment Setup (One-Time)

These steps are performed manually on your hosting account (cPanel and SSH Terminal).

### 1. Host Environment Check

Before proceeding, ensure your environment meets Laravel's requirements

**PHP Version**: Log into cPanel, go to "Select PHP Version," and choose a modern version (PHP 8.2 or higher ***Recommended***).

**Enable Extensions**: Ensure the following mandatory PHP extensions are enabled (look for them in the same PHP Version selector):

- mbstring
- pdo
- pdo_mysql
- json
- xml
- tokenizer

### 2. Set Up Deployment Path and Webroot

**Create Deployment Directory**:

- Log into your cPanel File Manager.
- Create a dedicated, private directory for your application files (e.g., laravelapp).
- Create a public folder in this directory (e.g., laravelapp/public).

The path will look like: /home2/username/public_html/laravelapp. This is your ***DEPLOY_PATH***.

**Point Webroot**:

- Go to Domains or Subdomains in cPanel.
- Ensure your domain or subdomain is pointing to the Laravel public directory: /home2/username/public_html/laravelapp/public. This tells the webserver where to start executing code.

### 3. Install Composer via cPanel Terminal

Shared hosting usually requires a manual, local installation of Composer.

**Connect via SSH/Terminal and run the following commands**:
```bash
# Move to the desired application directory (where the files will eventually live)
cd /home2/username/public_html/laravelapp
```
```bash
# Download the Composer installer script using curl.
# We use curl instead of PHP copy() because allow_url_fopen is disabled on the server.
curl -sS [https://getcomposer.org/installer](https://getcomposer.org/installer) -o composer-setup.php
```
```bash
# Fetch the latest official installer SHA384 hash for security verification.
HASH="$(curl -sS [https://composer.github.io/installer.sig](https://composer.github.io/installer.sig))"
```
```bash
# Verify that the downloaded installer matches the official signature.
php -r "if (hash_file('sha384', 'composer-setup.php') === '$HASH') { echo 'Installer verified', PHP_EOL; } else { echo 'Installer corrupt', PHP_EOL; unlink('composer-setup.php'); }"
```
```bash
# Create a local bin directory inside the user's home folder.
mkdir -p $HOME/bin
```
```bash
# Run the installer and force-enable allow_url_fopen only for this command.
# This bypasses the hosting restriction without modifying php.ini.
php -d allow_url_fopen=1 composer-setup.php --install-dir=$HOME/bin --filename=composer
```
```bash
# Add ~/bin to the user's PATH so "composer" can be executed from any directory.
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bash_profile
```
```bash
# Reload the updated profile so PATH changes take effect immediately.
source ~/.bash_profile
```
```bash
# Verify the Composer installation.
composer -V
```

### 4. Create and Configure .env File

The .env file should NEVER be deployed via Git/FTP for security reasons. It must be created once on the server.

First upload .env.example to the deployment root directory

**Run the following commands in the server terminal**:

```bash
# Move to the directory
cd /home2/username/public_html/laravelapp
```
```bash
# Create the .env from .env.example
cp .env.example .env
```
```bash
# Generate a new APP_KEY which updates the .env automatically
php artisan key:generate --force
```
In case the automatic APP_KEY generation fails use the below command
```bash
# Generate a new APP_KEY manually
# Copy the output (e.g., base64:...) and update the APP_KEY in .env manually
php -r "echo 'base64:'.base64_encode(random_bytes(32));"
```
Go ahead and update the other content of your .env file manually

## Phase 2: GitHub Deployment (FTP)
This sets up the file transfer and SSH step to execute all necessary Laravel commands.

### 1. Create FTP Account and Secrets
Create FTP Account: In cPanel, go to "FTP Accounts" and create a dedicated FTP user account, restricting its directory access to the deployment path (e.g., /home2/username/public_html/laravelapp).

**Add GitHub Secrets**: Go to your GitHub repository > Settings > Secrets and Variables > Actions, and add the following three secrets:

1. FTP_SERVER: The hostname or IP address (e.g., ftp.example.com).
2. FTP_USERNAME: The full FTP username.
3. FTP_PASSWORD: The FTP user's password.

### 2. Create and Authorize SSH Key
1. Generate Key: In cPanel, go to "SSH Access" or "SSH Keys."
   - Generate a new SSH key pair. It is highly recommended to use a passphrase for security.
3. Authorize Key: Once generated, ensure the public key is explicitly Authorized for use with your cPanel username.
4. Get Private Key: Download or copy the Private Key content, including the header and footer lines.

**Create Github SSH Secrets**: Go to your GitHub repository > Settings > Secrets > Actions, and add the following secrets
1. SSH_HOST: Your domain or server IP (e.g., example.com)
2. SSH_USER: Your cPanel/SSH username.
3. SSH_PRIVATE_KEY: The full content of the private key (including -----BEGIN...).
4. SSH_PASSPHRASE: The password you set for the private key.
5. DEPLOY_PATH: The absolute path on the server (e.g., /home2/username/public_html/laravelapp).

### 3. Deployment Script (.github/workflows/deploy.yml)
Do this locally. This script handles installation of composers dependencies, basic file transfer (the vendor/ folder and content will be created and uploaded to your deployment path automatically here so ensure to ignore it while pushing to Github) and execution of all necessary Laravel commands.

```bash
name: 🚀 Deploy Laravel App via FTP

on:
  push:
    branches:
      - main  # Deploy only when pushing to main branch

jobs:
  deploy:
    name: Upload Laravel app via FTP
    runs-on: ubuntu-latest

    steps:
      # 1️⃣ Checkout repository
      - name: Checkout repository
        uses: actions/checkout@v3

      # 2️⃣ Install Composer
      - name: Install Composer dependencies
        run: composer install --no-dev --optimize-autoloader

      # 3️⃣ Deploy via FTP
      - name: FTP Deploy to Go54
        uses: SamKirkland/FTP-Deploy-Action@v4.3.4
        with:
          server: ${{ secrets.FTP_SERVER }}
          username: ${{ secrets.FTP_USERNAME }}
          password: ${{ secrets.FTP_PASSWORD }}
          server-dir: ./ # Deploying to the root directory of the subdomain (specify if you are not allowed to specify in the ftp account you created)
          local-dir: ./   # repo root
          dangerous-clean-slate: false   # keep existing files not in repo
          exclude: |
            .git/**
            **/.git/**
            .github/**
            **/.github/**
            node_modules/**
            **/node_modules/**
            tests/**
            **/tests/**
            composer.json
            composer.lock
            README.md
            deploy.yml
            .env
            vendor/bin/**
            deploy.yml
            README.md
      # 4️⃣ Run Migrations and Optimize
      - name: Execute migrations and optimize cache via SSH
        uses: appleboy/ssh-action@v1.0.1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          passphrase: ${{ secrets.SSH_PASSPHRASE }}
          script: |
            # Navigate to the correct directory
            cd ${{ secrets.DEPLOY_PATH }}
            
            # Run migrations (use --force in production environments)
            php artisan migrate --force
            
            # Clear configuration and route caches for the new deployment
            php artisan config:cache
            php artisan route:cache
            php artisan view:cache

```

### 4. Push Laravel App to GitHub
Ensure your local Laravel project is committed and pushed to your GitHub repository on the main branch.

***Action**: Commit and push this file to GitHub. It will run and upload all files (including vendor/) to your server, connect via SSH, run migrations, and clear caches, making your application fully operational!*
