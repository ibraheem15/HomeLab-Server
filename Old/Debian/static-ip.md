# Static IP Configuration with Netplan

This guide explains how to configure a static IP address using Netplan on Ubuntu or similar systems.

https://netplan.readthedocs.io/en/stable/examples/#how-to-configure-a-static-ip-address-on-an-interface

## Configuration Steps
1. Install Netplan if not already installed:
  ```bash
  sudo apt update
  sudo apt install netplan.io
  ```

2. Create or edit the Netplan configuration file:
  ```bash
  sudo nano /etc/netplan/01-netcfg.yaml
  ```

3. Add the following configuration:
  ```yaml
  network:
    version: 2
    renderer: networkd
    ethernets:
     enp0s3:
      dhcp4: no
      dhcp6: no
      addresses:
        - 10.0.2.15/24
      routes: 
        - to: default
         via: 10.0.2.2
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
        search: []
  ```

4. Apply the configuration:
  ```bash
  sudo netplan apply
  ```

5. Verify the configuration:
  ```bash
  ip addr show enp0s3
  ```

## Configuration Details
- `enp0s3`: Network interface name
- `10.0.2.15/24`: Static IP address with subnet mask
- `10.0.2.2`: Default gateway
- DNS servers: Google (8.8.8.8) and Cloudflare (1.1.1.1)

**Note**: Make sure to replace `enp0s3` with your actual network interface name if different.

