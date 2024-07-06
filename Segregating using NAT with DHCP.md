Proxmox Networking Setup
========================

Overview
--------
This guide will assist you in setting up NAT to segregate your machines/containers on Proxmox. It covers the configuration of a new Linux bridge network, enabling NAT, configuring DHCP, and connecting clients to the NAT.

Network Topology
----------------
- **Physical Network**: One network card attached to the server (built-in) and connected directly to home router.
- **Proxmox Server**: The server hosts a running DNS server and a WireGuard VPN server. A new web-facing server for a personal website will be created and placed in the new NAT to segregate it from the rest of the network.

Steps for Creating a New NAT on Proxmox
----------------------------------------
1. Navigate to the network section under the Proxmox server.
2. Create a new Linux bridge.
3. Assign a name, an IP address, and CIDR to the network (e.g., "192.168.100.1/24", "vmbr1").
4. Access the Proxmox console.
5. Open the `/etc/network/interfaces` file (`nano /etc/network/interfaces`).
6. Add the following configuration:

    ```bash
    auto vmbr1
    iface vmbr1 inet static
        address 192.168.100.1
        netmask 255.255.255.0
        bridge-ports none
        bridge-stp off
        bridge-fd 0
    ```

7. Exit the editor.
8. Allow IPv4 forwarding: `sysctl -w net.ipv4.ip_forward=1`.
9. Reload sysctl settings: `sysctl -p`.
10. Add NAT rules (192.168.2.0/24 is the vmbr0 network):

    ```bash
    iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o vmbr0 -j MASQUERADE
    iptables -A FORWARD -s 192.168.100.0/24 -d 192.168.2.0/24 -j DROP
    ```

11. Make config persistent:

    ```bash
    apt-get update
    apt-get install iptables-persistent

    iptables-save > /etc/iptables/rules.v4
    ip6tables-save > /etc/iptables/rules.v6
    ```

12. Check the rules: `iptables -t nat -L` - you should be able to see the new rules.

Configuring DHCP for the NAT
-----------------------------
1. Install ISC DHCP server:

    ```bash
    apt-get update
    apt-get install isc-dhcp-server
    ```

2. Edit the DHCP server configuration file:

    ```bash
    nano /etc/dhcp/dhcpd.conf
    ```

3. Add the following configuration:

    ```bash
    subnet 192.168.100.0 netmask 255.255.255.0 {
        range 192.168.100.100 192.168.100.200;
        option routers 192.168.100.1;
        option domain-name-servers <dns-server-ip>;
    }
    ```

4. Start and enable the DHCP service:

    ```bash
    systemctl start isc-dhcp-server
    systemctl enable isc-dhcp-server
    ```

Adding Clients to the NAT
-------------------------
1. Navigate to the server you want to add to the NAT.
2. Open the network tab and select the new network (`vmbr1`).
3. Enable DHCP and check connectivity.

Test connectivity
-------------------------
![alt text](screenshots/image1.png)

Troubleshooting
---------------
- If you encounter the following error when starting/restarting isc-dhcp:

    ```
    root@proxmox:~# systemctl restart isc-dhcp-server
    Job for isc-dhcp-server.service failed because the control process exited with error code.
    See "systemctl status isc-dhcp-server.service" and "journalctl -xe" for details.
    ```

    - Add the  bridge `vmbr1` to the `/etc/default/isc-dhcp-server` file:

    ```bash
    # The default bridge is vmbr0.
    INTERFACES="vmbr1"
    ```

    - Restart isc-dhcp again:

    ```bash
    root@proxmox:~# systemctl restart isc-dhcp-server
    ```

- Review the steps outlined above to troubleshoot and resolve any other issues you might encounter.
