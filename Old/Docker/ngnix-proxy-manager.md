# Nginx Proxy Manager Setup with Portainer

### https://nginxproxymanager.com/guide/#quick-setup

## Prerequisites
- Docker installed
- Portainer installed and running

## Steps to Deploy

1. **Create New Stack in Portainer**
  - Navigate to Stacks in Portainer
  - Click "Add stack"
  - Name your stack (e.g., "nginx-proxy-manager")

2. **Docker Compose Configuration**
  ```yaml
  version: '3'
  services:
    npm:
     image: 'jc21/nginx-proxy-manager:latest'
     restart: unless-stopped
     ports:
      - '80:80'
      - '81:81'
      - '443:443'
     volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
  ```

3. **Default Login Credentials**
  - URL: `http://your-ip:81`
  - Email: `admin@example.com`
  - Password: `changeme`

4. **First Time Setup**
  - Login with default credentials
  - Change the default password
  - Start configuring your proxy hosts

## Notes
- Port 80: HTTP traffic
- Port 81: Admin interface
- Port 443: HTTPS traffic