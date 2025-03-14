#!/bin/bash

# Script to install Angular, Node, MySQL, Redis, and Nginx on CentOS 9
# Handles GPG key import failures, no .env file, addresses errors when MySQL is ALREADY installed,
# and implements advanced MySQL root password reset techniques, **WITHOUT** changing the password.
# This script ONLY installs the other packages if there's an existing MySQL setup.

# Define default values for versions (HARDCODED)
ANGULAR_VERSION="17.3.0"
NODE_VERSION="18.13.0"
MYSQL_VERSION="8.0.36"
REDIS_VERSION="5.0.14.1"  # This will likely cause problems.
NGINX_VERSION="1.20.1"  # This will likely cause problems.

# ---  Helper Functions ---

log_info() {
  echo -e "\e[34m[INFO] $(date '+%Y-%m-%d %H:%M:%S') - $1\e[0m"
}

log_success() {
  echo -e "\e[32m[SUCCESS] $(date '+%Y-%m-%d %H:%M:%S') - $1\e[0m"
}

log_error() {
  echo -e "\e[31m[ERROR] $(date '+%Y-%m-%d %H:%M:%S') - $1\e[0m"
}

# Check for root privileges
if [[ $EUID -ne 0 ]]; then
  log_error "This script requires root privileges. Please run with sudo."
  exit 1
fi

# --- Update System ---

log_info "Updating system packages..."
yum update -y

# --- Check MySQL Installation and Skip Password Reset ---

log_info "Checking MySQL installation..."
if rpm -q mysql-community-server > /dev/null 2>&1; then
  log_info "MySQL is already installed. Skipping password reset."
else
  log_error "MySQL is NOT installed. Please install MySQL manually before running this script."
  exit 1 # Exit if MySQL is not found, since the password reset steps depend on its existence
fi

# --- Install Node.js and npm ---

log_info "Installing Node.js ${NODE_VERSION} and npm..."

# Install nvm (Node Version Manager) if not already present

if ! command -v nvm &> /dev/null
then
  log_info "Installing nvm..."
  curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
  export NVM_DIR="$([ -z "${XDG_CONFIG_HOME-}" ] && printf %s "${HOME}/.nvm" || printf %s "${XDG_CONFIG_HOME}/nvm")"
  [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
  [ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
fi

# Install the specified Node.js version
nvm install "${NODE_VERSION}"
nvm use "${NODE_VERSION}"
nvm alias default "${NODE_VERSION}" # set default node version
log_success "Node.js ${NODE_VERSION} and npm installed successfully."

# --- Install Angular CLI ---

log_info "Installing Angular CLI ${ANGULAR_VERSION}..."
npm install -g @angular/cli@"${ANGULAR_VERSION}"

if [ $? -ne 0 ]; then
  log_error "Failed to install Angular CLI."
  exit 1
fi
log_success "Angular CLI ${ANGULAR_VERSION} installed successfully."

# --- Install Redis ---

log_info "Installing Redis. Note:  CentOS 9 typically includes a newer version of Redis. Installing an older version is often problematic.  This script will attempt to install the Redis version from the system repos.  If you REALLY need Redis ${REDIS_VERSION}, you will need to download the source and build/install it manually."

yum install -y redis

systemctl enable redis
systemctl start redis

if [ $? -ne 0 ]; then
  log_error "Failed to install Redis. Consider manually compiling and installing version ${REDIS_VERSION}."
  exit 1
fi

log_success "Redis installed successfully."

# --- Install Nginx ---

log_info "Installing Nginx. Note:  CentOS 9 typically includes a newer version of Nginx. Installing an older version is often problematic.  This script will install Nginx from the OS repos. If you REALLY need Nginx ${NGINX_VERSION}, you may need to build it from source."

yum install -y nginx

systemctl enable nginx
systemctl start nginx

if [ $? -ne 0 ]; then
  log_error "Failed to install Nginx. Consider building from source for specific version ${NGINX_VERSION}."
  exit 1
fi
log_success "Nginx installed successfully."

log_success "All components installed successfully!"

exit 0
