# Raspberry Pi - MANET Network Setup

## Table of Contents
- [Introduction](#introduction)
- [Network Architecture](#network-architecture)
- [Prerequisites](#prerequisites)
- [Manual Setup](#manual-setup)
  - [Step 1: Enable Wi-Fi Ad Hoc Mode](#step-1-enable-wi-fi-ad-hoc-mode)
  - [Step 2: Set Up BATMAN-ADV](#step-2-set-up-batman-adv)
  - [Step 3: Verify MANET Network](#step-3-verify-manet-network)
  - [Step 4: Enable Name-Based Discovery (Avahi)](#step-4-enable-name-based-discovery-avahi)
  - [Step 5: Make Configuration Persistent](#step-5-make-configuration-persistent)
- [Automated Setup with Ansible](#automated-setup-with-ansible)
  - [Preparing the Control Machine](#preparing-the-control-machine)
  - [Key Ansible Playbooks](#key-ansible-playbooks)
  - [Running the Playbooks](#running-the-playbooks)
- [Troubleshooting](#troubleshooting)
- [Performance Optimization](#performance-optimization)
- [Future Extensions](#future-extensions)
- [Resources and References](#resources-and-references)

## Introduction

A Mobile Ad Hoc Network (MANET) is a self-configuring, infrastructure-less network of mobile devices connected wirelessly. Each device in a MANET is free to move independently in any direction, and will therefore change its links to other devices frequently.

This project implements a MANET using Raspberry Pi devices and the BATMAN-ADV (Better Approach To Mobile Ad-hoc Networking) protocol, which is a layer 2 routing protocol designed for ad-hoc networks. BATMAN-ADV handles all the routing complexity in the background, making it ideal for dynamic networks where nodes may join, leave, or move around.

**Key Benefits of This Setup:**
- **Resilient Communication**: Creates a self-healing mesh network that can route around node failures
- **No Single Point of Failure**: Fully distributed architecture with no central controller
- **Easy Scaling**: Seamlessly add more nodes to expand coverage
- **Works Without Infrastructure**: Functions in environments without existing networks
- **Device Discovery**: Automatic service discovery via Avahi

## Network Architecture


The BATMAN-ADV protocol automatically builds and maintains routing tables to find optimal paths between all nodes, even when they cannot directly communicate with each other.

## Prerequisites

Before starting, ensure you have:

- A Raspberry Pi Zero 2W, 4 or 5 with Ubuntu 22.04 Server (arm64), or newer installed on all devices
- SSH access enabled or a monitor and keyboard attached
- A working internet connection (for package installation)
- Basic knowledge of Linux commands

Note: Raspberry Pi Zero 2W may require different system (32bit version), as the support for 64bit version has been stopped.

## Manual Setup

This section provides step-by-step instructions for manually configuring each Raspberry Pi. For automated setup, skip to the [Ansible section](#automated-setup-with-ansible).

### Step 1: Enable Wi-Fi Ad Hoc Mode

These steps must be followed for each Raspberry Pi that will be connected to the network. Remember to update each node name or node IP address for each node.

1. **Update and upgrade packages**:

    ```bash
    sudo apt update && sudo apt upgrade -y
    ```

2. **Install required packages**:

    ```bash
    sudo apt install batctl iw net-tools avahi-daemon avahi-utils -y
    ```

3. **Check the name of your wireless interface**:

    ```bash
    iw dev
    ```

    Example output:

    ```bash
    Interface wlan0
        ifindex 3
        wdev 0x1
        addr xx:xx:xx:xx:xx:xx
        type managed
    ```

    Note that `wlan0` is the name of your wireless interface.

4. **Disable and reconfigure the Wi-Fi interface**:

    We need to set our wireless interface to work in IBSS mode, which supports Ad-Hoc networks.

    ```bash
    sudo ip link set wlan0 down
    sudo iw dev wlan0 set type ibss
    sudo ip link set wlan0 up
    ```

5. **Create an Ad Hoc network**:

    ```bash
    sudo iw dev wlan0 ibss join "MANET-NET" 2412
    ```

    - `MANET-NET`: The name of the ad hoc network
    - `2412`: Channel frequency (2.4GHz, Channel 1). Both RPI 4 and 5 support Wi-Fi 5GHz, so this can also be utilized

6. **Verify network type change**:

    ```bash
    iw dev
    ```

    You should see something like:

    ```bash
    Interface wlan0
        ifindex 3
        wdev 0x1
        addr xx:xx:xx:xx:xx:xx
        ssid MANET-NET
        type IBSS
        channel 1 (2412 MHz), width: 20 MHz, center1: 2412 MHz
        txpower 31.00 dBm
    ```

### Step 2: Set Up BATMAN-ADV

1. **Load the BATMAN-ADV kernel module**:

    ```bash
    sudo modprobe batman-adv
    ```

    To ensure it loads on boot:

    ```bash
    echo "batman-adv" | sudo tee -a /etc/modules
    ```

2. **Attach Wi-Fi to BATMAN-ADV**:

    ```bash
    sudo ip link set wlan0 mtu 1500
    sudo batctl if add wlan0
    sudo ip link set bat0 up
    ```

3. **Assign an IP address**:

    ```bash
    sudo ip addr add 172.0.10.1/24 dev bat0
    ```

    Replace `172.0.10.1` with a unique IP per node, following the pattern `172.0.10.X` where X is a unique number for each node.

### Step 3: Verify MANET Network

1. **Check connected nodes**:

    ```bash
    sudo batctl n
    ```

    You should see something like this:

    ```bash
    [B.A.T.M.A.N. adv 2024.2, MainIF/MAC: wlan0/xx:xx:xx:xx:xx:xx (bat0/xx:xx:xx:xx:xx:xx BATMAN_IV)]
    IF             Neighbor              last-seen
        wlan0     xx:xx:xx:xx:xx:xx    0.709s
        wlan0     xx:xx:xx:xx:xx:xx    0.178s
    ```

2. **Check routing table**:

    ```bash
    sudo batctl o
    ```

    You should see something like this:

    ```bash
    [B.A.T.M.A.N. adv 2024.2, MainIF/MAC: wlan0/xx:xx:xx:xx:xx:xx (bat0/xx:xx:xx:xx:xx:xx BATMAN_IV)]
    Originator                  last-seen   (#/255)     Nexthop             [outgoingIF]
        xx:xx:xx:xx:xx:xx       0.813s      (198)       xx:xx:xx:xx:xx:xx   [     wlan0]
        * xx:xx:xx:xx:xx:xx     0.813s      (247)       xx:xx:xx:xx:xx:xx   [     wlan0]
        xx:xx:xx:xx:xx:xx       0.320s      (191)       xx:xx:xx:xx:xx:xx   [     wlan0]
        * xx:xx:xx:xx:xx:xx     0.320s      (255)       xx:xx:xx:xx:xx:xx   [     wlan0]
    ```

3. **Ping another node**:

    ```bash
    ping 172.0.10.2
    ```

    You should see something like this:

    ```bash
    PING 172.0.10.2 (172.0.10.2) 56(84) bytes of data.
    64 bytes from 172.0.10.2: icmp_seq=1 ttl=64 time=7.58 ms
    64 bytes from 172.0.10.2: icmp_seq=2 ttl=64 time=8.06 ms
    64 bytes from 172.0.10.2: icmp_seq=3 ttl=64 time=1.29 ms
    64 bytes from 172.0.10.2: icmp_seq=4 ttl=64 time=7.38 ms
    ^C
    --- 172.0.10.2 ping statistics ---
    4 packets transmitted, 4 received, 0% packet loss, time 3005ms
    rtt min/avg/max/mdev = 1.291/6.079/8.063/2.775 ms
    ```

### Step 4: Enable Name-Based Discovery (Avahi)

Avahi provides service discovery capabilities, allowing nodes to find each other by name instead of IP address.

1. **Modify Avahi daemon configuration**:

    ```bash
    sudo nano /etc/avahi/avahi-daemon.conf
    ```

    Update the file as follows:

    ```conf
    [server]
    host-name=rpi-node
    domain-name=manet-local
    use-ipv4=yes
    use-ipv6=no
    allow-interfaces=bat0
    ratelimit-interval-usec=1000000
    ratelimit-burst=1000

    [wide-area]
    enable-wide-area=no

    [publish]
    publish-addresses=yes
    publish-hinfo=yes
    publish-workstation=yes
    publish-domain=yes

    [reflector]
    enable-reflector=yes
    ```

    Replace `rpi-node` with a specific name for each node (e.g., `rpi-node1`, `rpi-node2`).
    
    Note: We're using `bat0` as the interface instead of `wlan0` to ensure Avahi operates on the BATMAN-ADV virtual interface.

2. **Restart Avahi service**:

    ```bash
    sudo systemctl restart avahi-daemon
    sudo systemctl enable avahi-daemon
    ```

3. **Verify Avahi is running**:

    ```bash
    sudo systemctl status avahi-daemon
    ```

4. **Discover other nodes**:

    ```bash
    avahi-browse -a
    ```

    You should see something like this:

    ```bash
    +  bat0 IPv4 rpi-node2 [xx:xx:xx:xx:xx:xx]                    Workstation          manet-local
    +  bat0 IPv4 rpi-node1 [xx:xx:xx:xx:xx:xx]                    Workstation          manet-local
    ```

### Step 5: Make Configuration Persistent

To make the MANET setup persist across reboots:

1. **Create configuration file**:

    ```bash
    sudo nano /etc/network/interfaces.d/adhoc-mesh
    ```

    Add the following:

    ```bash
    auto wlan0
    iface wlan0 inet manual
    pre-up ip link set wlan0 down
    pre-up iw dev wlan0 set type ibss
    pre-up ip link set wlan0 up
    pre-up iw dev wlan0 ibss join "MANET-NET" 2412
    pre-up ip link set wlan0 mtu 1500
    pre-up batctl if add wlan0
    up ip link set bat0 up
    up ip addr add 172.0.10.1/24 dev bat0
    ```

    Remember to replace `172.0.10.1` with the unique IP for each node.

2. **Create a systemd service for batman-adv**:

    ```bash
    sudo nano /etc/systemd/system/batman-adhoc.service
    ```

    Add the following:

    ```ini
    [Unit]
    Description=Configure batman-adv ad hoc network
    After=network.target

    [Service]
    Type=oneshot
    ExecStart=/bin/bash -c 'modprobe batman-adv && ip link set wlan0 down && iw dev wlan0 set type ibss && ip link set wlan0 up && iw dev wlan0 ibss join "MANET-NET" 2412 && ip link set wlan0 mtu 1500 && batctl if add wlan0 && ip link set bat0 up && ip addr add 172.0.10.1/24 dev bat0'
    RemainAfterExit=yes

    [Install]
    WantedBy=multi-user.target
    ```

    Again, replace `172.0.10.1` with the unique IP for each node.

3. **Enable the service**:

    ```bash
    sudo systemctl enable batman-adhoc
    ```

## Automated Setup with Ansible

This repository includes Ansible playbooks to automate the setup process across multiple Raspberry Pi devices. This is the recommended approach for setting up multiple nodes.

### Preparing the Control Machine

1. **Install Ansible on your control machine**:

    ```bash
    sudo apt update
    sudo apt install ansible -y
    ```

2. **Clone this repository**:

    ```bash
    git clone https://github.com/yourusername/rpi-manet.git
    cd rpi-manet
    ```

3. **Update the hosts.yml file** with your Raspberry Pi IPs and the desired MANET IPs:

    ```yaml
    all:
      vars:
        ansible_user: yourusername
        ansible_ssh_private_key_file: ~/.ssh/id_rsa_manet
        adhoc_ssid: "rpi-adhoc"
        adhoc_freq: "2412"
        batman_interface: "bat0"
        manet_netmask: "24"

      hosts:
        rpi5-1:
          ansible_host: 192.168.40.1
          manet_ip: 172.0.10.1
        rpi5-2:
          ansible_host: 192.168.40.2
          manet_ip: 172.0.10.2
        # Add more nodes as needed
    ```

### Key Ansible Playbooks

The repository includes several playbooks:

1. **distribute-ssh-keys.yml**: Sets up SSH keys for passwordless login to all nodes
2. **update.yml**: Updates all packages on the Raspberry Pi nodes
3. **setup-batman.yml**: Configures the BATMAN-ADV network
4. **setup-avahi.yml**: Sets up Avahi for service discovery
5. **check-connectivity-manet.yml**: Tests connectivity between nodes
6. **reboot.yml**: Reboots all nodes
7. **shutdown.yml**: Shuts down all nodes

### Running the Playbooks

1. **First, distribute SSH keys**:

    ```bash
    ansible-playbook playbooks/distribute-ssh-keys.yml
    ```

2. **Update all nodes**:

    ```bash
    ansible-playbook playbooks/update.yml
    ```

3. **Set up the BATMAN-ADV network**:

    ```bash
    ansible-playbook playbooks/setup-batman.yml
    ```

4. **Configure Avahi service discovery**:

    ```bash
    ansible-playbook playbooks/setup-avahi.yml
    ```

5. **Test connectivity between nodes**:

    ```bash
    ansible-playbook playbooks/check-connectivity-manet.yml
    ```

## Troubleshooting

### Common Issues and Solutions

1. **Nodes can't see each other**:
   - Ensure all nodes are using the same SSID and frequency
   - Check that wireless interfaces are in IBSS mode (`iw dev`)
   - Verify BATMAN-ADV is running (`batctl n`)
   - Try moving nodes closer together temporarily

2. **Network stops working after reboot**:
   - Check if the systemd service is enabled and running
   - Verify logs with `journalctl -u batman-adhoc`
   - Ensure the batman-adv module is loaded (`lsmod | grep batman`)

3. **Poor performance or high latency**:
   - Try different Wi-Fi channels to avoid interference
   - Increase MTU if network conditions allow
   - Consider strategic placement of nodes for better signal

4. **Avahi service discovery not working**:
   - Check if the Avahi daemon is running (`systemctl status avahi-daemon`)
   - Verify it's configured to use the `bat0` interface
   - Check firewall settings to ensure mDNS traffic is allowed

### Diagnostic Commands

```bash
# Check batman-adv kernel module
lsmod | grep batman

# View batman-adv debug logs
batctl l

# Monitor batman-adv mesh status
watch -n1 "batctl n"

# Test Avahi discovery
avahi-browse -a

# Check IP configuration
ip addr show bat0
```

## Performance Optimization

1. **Wi-Fi Channel Selection**:
   - Use less congested channels (check with `iwlist wlan0 scan`)
   - Consider 5GHz if available (less interference)

2. **Power Management**:
   - Disable Wi-Fi power management for better responsiveness:
     ```bash
     sudo iw dev wlan0 set power_save off
     ```

3. **Transmission Power**:
   - Increase TX power if allowed in your region:
     ```bash
     sudo iw dev wlan0 set txpower fixed 3000
     ```

4. **Routing Algorithm Optimization**:
   - Adjust BATMAN-ADV parameters for your specific use case:
     ```bash
     sudo batctl gw_mode server
     sudo batctl hop_penalty 30
     ```

## Future Extensions

### 1. Advanced Security Features

- **VPN Implementation**:
  - Set up WireGuard or OpenVPN tunnels between nodes
  - Encrypt all traffic in the mesh network
  - Example implementation available in `security-enhancements` branch

- **Firewall Rules**:
  - Configure `iptables` or `nftables` rules to restrict access
  - Create segmented network zones for different types of traffic

- **Intrusion Detection**:
  - Deploy lightweight IDS like Zeek or Suricata for anomaly detection
  - Set up log forwarding to a central monitoring node

### 2. Network Visualization

- **Real-time Monitoring Dashboard**:
  - Deploy Grafana + Prometheus for network metrics
  - Visualize bandwidth, latency, and packet loss
  - See example implementation in the `monitoring` branch

- **Topology Visualization**:
  - Use d3.js or Vis.js to create dynamic network maps
  - Show node connections and signal strength

### 3. Automated Node Configuration

- **Dynamic IP Allocation**:
  - Implement lightweight DHCP service or DHT-based addressing
  - Avoid manual IP assignment for new nodes

- **Configuration Management**:
  - Use Ansible for centralized configuration updates
  - Create automated provisioning workflows

### 4. Service Hosting

- **Distributed Applications**:
  - Deploy chat servers, file sharing, or collaborative tools
  - Set up service discovery using Avahi and DNS-SD

- **Containerized Services**:
  - Use lightweight Docker containers for services
  - Deploy MicroK8s for container orchestration

## Resources and References

- [BATMAN-ADV Protocol Documentation](https://www.open-mesh.org/projects/batman-adv/wiki)
- [Raspberry Pi Networking Guides](https://www.raspberrypi.org/documentation/computers/networking.html)
- [Avahi Service Discovery](https://www.avahi.org/doxygen/html/index.html)
- [Ansible Documentation](https://docs.ansible.com/)
- [Linux Wireless Subsystem](https://wireless.wiki.kernel.org/)
