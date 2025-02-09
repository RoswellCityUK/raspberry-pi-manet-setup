# Raspberry Pi - MANET Network Setup

## Prerequisites

Before starting, ensure you have:

- A Raspberry Pi 4 or 5 with Ubuntu 22.04 Server (arm64), or newer installed on all devices.
- SSH access enabled or a monitor and keyboard attached.
- A working internet connection (for package installation).
- Basic knowledge of Linux commands.

## Step 1: Enable Wi-Fi Ad Hoc Mode

This steps have to be followed for each Raspberry PI that will be connected to the network. Remember to update each node name or node IP address for each node.

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

    We need to set our wireless interface to work in IBSS mode, which support Ad-Hoc networks.

    ```bash
    sudo ip link set wlan0 down
    sudo iw dev wlan0 set type ibss
    sudo ip link set wlan0 up
    ```

5. **Create an Ad Hoc network**:

    ```bash
    sudo iw dev wlan0 ibss join "MANET-NET" 2412
    ```

    - `MANET-NET`: The name of the ad hoc network.
    - `2412`: Channel frequency (2.4GHz, Channel 1). Both RPI 4 and 5 supports Wi-Fi 5GHz, so this can be also utilized.

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

## Step 2: Set Up BATMAN-ADV

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

    Replace `172.0.10.1` with a unique IP per node.

## Step 3: Verify MANET Network

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

## Step 4: Enable Name-Based Discovery (Avahi)

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
    allow-interfaces=wlan0
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

    Replace `rpi-node` - hostname, with specific name for each node.
    You can change `use-ipv6` to `yes` if you want to use IPv6 in your network.

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
    +  wlan0 IPv4 rpi5-2 [xx:xx:xx:xx:xx:xx]                    Workstation          manet-local
    +  wlan0 IPv4 rpi5-1 [xx:xx:xx:xx:xx:xx]                    Workstation          manet-local
    ```

## Step 5: Setup persistent across reboots (Optional)

To make the MANET setup persistent across reboots:

1. **Edit `/etc/network/interfaces.d/adhoc-mesh`**:

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

2. **Save and exit**.

3. **Make the script executable**:

    ```bash
    sudo chmod +x /etc/network/interfaces.d/adhoc-mesh
    ```

## Possible extensions in the future

1. Adding advanced security features to the network:
   - Implement VPN tunnels for encrypted communication.
   - Use firewall rules to restrict access.
   - Deploy intrusion detection systems (IDS) for anomaly detection.
2. Visualization of the network
   - Develop a web-based dashboard to monitor network traffic and topology.
   - Use tools like Grafana, Prometheus, or Vis.js for real-time visualization.
3. Automated Node Configuration
   - Create a script to automate network configuration on new nodes - Ansible.
   - Implement dynamic IP allocation using a lightweight DHCP service.
4. Optimized Performance
   - Experiment with different Wi-Fi channels to reduce interference.
   - Configure Quality of Service (QoS) settings to prioritize critical data.
   - Test different routing protocols for better performance.
5. Service Hosting on MANET
   - Deploy distributed applications like chat servers or file sharing.
   - Run containerized applications (Docker) for lightweight services.
   - Integrate solution similar to MicroK8s clusters to deploy services across network.

## Conclusion

Following this guide, you can set up a Raspberry Pi-based MANET using BATMAN-ADV and Avahi for decentralized networking. You can expand on this by adding services such as a distributed file system or sensor data sharing.

---

✅ **Test each step before moving on.**
✅ **Make configuration changes persistent.**
✅ **Experiment with different channels to optimize connectivity.**
