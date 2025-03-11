# Nextcloud Vagrant Setup

This repository provides an automated Vagrant setup for deploying a Nextcloud instance on a Debian Bookworm-based virtual machine.

## Features
- Runs on Debian Bookworm (64-bit)
- Apache2 web server with SSL (self-signed certificate)
- MariaDB as the database server
- Redis caching for improved performance
- Fail2Ban for security protection
- UFW (Uncomplicated Firewall) for enhanced security
- Automatic installation and configuration of Nextcloud
- PHP optimizations for better performance
- Swap file creation for improved stability
- Systemd service for Nextcloud background jobs

## Prerequisites
Ensure that you have the following installed on your host machine:
- [Vagrant](https://www.vagrantup.com/)
- [VirtualBox](https://www.virtualbox.org/)

## Installation & Usage

### 1. Clone the Repository
```sh
git clone https://github.com/fthomys/nextcloud-vagrant.git
cd nextcloud-vagrant
```

### 2. Start the Virtual Machine
```sh
vagrant up
```
This command will start the VM and automatically provision it with the necessary configurations and installations.

### 3. Access Nextcloud
Once the setup is complete, Nextcloud will be accessible via:
- **HTTPS**: `https://localhost:8443`
- **HTTP** (redirects to HTTPS): `http://localhost:8080`

### 4. Login Credentials
After provisioning, the login credentials for Nextcloud and the database are stored in the VM at:
```sh
/var/nclogin.txt
```
To retrieve them:
```sh
vagrant ssh
cat /var/nclogin.txt
```
The initial Nextcloud admin credentials are also displayed in the console output during provisioning.

### 5. Stopping and Destroying the VM
To stop the VM:
```sh
vagrant halt
```
To completely remove the VM:
```sh
vagrant destroy
```

## Configuration Details
- **Apache Virtual Hosts** are configured to serve Nextcloud over HTTPS
- **MariaDB** is used as the database backend
- **Redis** is enabled for caching and locking
- **Fail2Ban** monitors Nextcloud logs for potential attacks
- **UFW** allows traffic only on ports 80, 443, and 22
- **A swap file (2GB)** is created to prevent memory issues
- **PHP 8.2 optimizations** are applied for better performance
- **Systemd service** is set up for Nextcloud background tasks

## Troubleshooting
- If Nextcloud is not reachable, check the VM’s status:
  ```sh
  vagrant status
  ```
- To debug potential issues inside the VM:
  ```sh
  vagrant ssh
  journalctl -u apache2 --no-pager
  ```
- If provisioning fails, try reloading and re-provisioning:
  ```sh
  vagrant reload --provision
  ```

## License
This project is licensed under the MIT License.

---
Enjoy your private Nextcloud instance!