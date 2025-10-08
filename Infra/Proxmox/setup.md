# Setting up a new Proxmox VE server

This guide will walk you through the steps to set up a new Proxmox VE server, including initial configuration, networking, storage, and security settings.

**Note: This guide is incomplete**

## Initial Setup

1. **Install Proxmox VE**: Download the latest Proxmox VE ISO from the [official website](https://www.proxmox.com/en/downloads) and create a bootable USB drive. Boot from the USB and follow the installation prompts.
2. **Access the Web Interface**: After installation, access the Proxmox web interface by navigating to `https://<your-server-ip>:8006` in your web browser. Log in using the root password you set during installation.
3. **Update Proxmox VE**: It's important to keep your Proxmox VE installation up to date. You can update the system via the web interface or by running the following commands in the terminal:
   ```bash
   apt update && apt dist-upgrade -y
   ```
4. **Set Up Networking**: Configure your network settings according to your environment. This may include setting up bridges for virtual machines and containers.
5. **Configure Storage**: Add and configure storage options such as local disks, NFS, or iSCSI targets for your virtual machines and containers.
6. **Create a Backup Strategy**: Set up a backup strategy to ensure your data is safe. Proxmox VE has built-in backup tools that you can configure via the web interface.
7. **Set Up User Accounts**: Create additional user accounts with appropriate permissions for managing the Proxmox VE server.
8. **Enable Two-Factor Authentication (2FA)**: For enhanced security, enable 2FA for user accounts via the web interface.
9. **Install Additional Packages**: Depending on your needs, you may want to install additional packages or tools to enhance the functionality of your Proxmox VE server.
10. **Monitor System Health**: Regularly monitor the health and performance of your Proxmox VE server using built-in tools or third-party monitoring solutions.
11. **Documentation and Support**: Familiarize yourself with the [Proxmox VE documentation](https://pve.proxmox.com/pve-docs/) and consider joining the Proxmox community forums for support and advice.

## Setup steps for hosting on Hetzner Dedicated Server

- Proxmox version: 9.x
- Host OS: Hetzner Rescue System

Culprits:
- Proxmox has to be installed via the rescue system which uses the `installimage` utility to install the OS. This utility only supports Proxmox VE 7.x and 8.x. We need to upgrade to Proxmox VE 9.x after the installation.
- Hetzner has strict MAC address policy - only the MAC addresses which are assigned to the server in Hetzner Robot are allowed to be used, otherwise the server will get a violation and may be disconnected. For this reason, we need to create a bridge network with a VNET in Proxmox

### Steps

1. Boot into the Hetzner Rescue System (Linux 64-bit).
2. Install Proxmox VE using the `installimage` utility:
    ```bash
    installimage
    ```
3. Follow the prompts to select Proxmox VE 7.x or 8.x and complete the installation.
4. Reboot into the newly installed Proxmox VE system.

[TODO: ADD INSTALLATION SCREENSHOTS]

5. Upgrade Proxmox VE to the latest 9.x version:
    ```bash
    apt update && apt dist-upgrade -y
    ```
6. Reboot the system to apply the upgrade.
7. Run the following commands to configure the instance:

    - basic configuration:
        ```bash
        passwd # set root password
        iptables -A INPUT -p tcp -m tcp --dport 111 -j DROP
        iptables -A INPUT -p udp -m udp --dport 111 -j DROP
        /sbin/iptables-save
        systemctl disable --now rpcbind rpcbind.socket
        systemctl mask rpcbind

        # Enable IP forwarding
        sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
        sed -i 's/#net.ipv6.conf.all.forwarding=1/net.ipv6.conf.all.forwarding=1/' /etc/sysctl.conf
        sysctl -p
        sysctl net.ipv4.ip_forward
        vi /etc/network/interfaces
        ```

    - network configuration:
        ```
        # /etc/network/interfaces
        source /etc/network/interfaces.d/*

        auto lo
        iface lo inet loopback

        iface lo inet6 loopback

        # ORIGINAL INTERFACE - COMMENT OUT
        #auto enp5s0
        #iface enp5s0 inet static
        #       address 94.13.88.63/26
        #       gateway 94.13.88.1
                #up route add -net 94.13.88.0 netmask 255.255.255.192 gw 94.13.88.1 dev enp5s0
                # route 94.13.88.0/26 via 94.13.88.1

        # ADD THIS
        iface enp5s0 inet manual

        # COMMENT IPV6 CONFIGURATION (we will use only ipv4 - configure ipv6 bridge if desired)
        #iface enp5s0 inet6 static
        #       address 2a01:4f9:3080:1928::2/64
        #       gateway fe80::1

        # ADD THIS - the actual bridge interface
        auto vmbr0 
        iface vmbr0 inet static
                address 94.13.88.63/26 # the IP of the original interface
                gateway 94.13.88.1 # the gateway of the original interface
                bridge-ports enp5s0 # the original interface
                bridge-stp off
                bridge-fd 0
        ```

    - setup VNET in proxmox
        [TODO: ADD SCREENSHOTS]

    - continue network configuration:
        ```bash
        systemctl restart networking
        ip a
        ping
        apt update
        apt -y install dnsmasq
        systemctl disable --now dnsmasq
        
        # add NAT rules - HTTP and HTTPS to internal IPs of the vnet
        iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 80 -j DNAT --to-destination 10.13.37.100:80
        iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 443 -j DNAT --to-destination 10.13.37.100:443

        /sbin/iptables-save
        ```
