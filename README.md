# Home Lab Project: Building a Multi-Hommed Router and Firewall with Debian

### **Project Summary**

This project focuses on building a secure virtual network environment to establish a segmented network and a centralized log collection system. The primary objective was to configure a Debian server to act as a central router and firewall for two isolated networks: one for a Windows Server and another for a Kali Linux machine. This setup enables hands-on experience in network administration, log analysis, and responding to simulated security events.

---

### **Table of Contents**

- [Topology Diagram](#topology-diagram)
- [Lab Components](#lab-components)
- [Configuration Guide](#configuration-guide)
- [Results and Verification](#results-and-verification)
- [Next Steps](#next-steps)

---

### **Topology Diagram**

The network was segmented to simulate a real-world environment where an external threat actor attempts to gain a foothold into the internal network.

<img width="1954" height="891" alt="home-lab-network" src="https://github.com/user-attachments/assets/da24706c-f177-4774-845b-e942185b1942" />
---

### **Lab Components**

* **Hypervisor:** (VirtualBox)
* **Operating Systems:**
    * Debian Gnu/Linux 13 (trixie)
    * Windows Server 2022
    * Kali GNU/Linux Rolling
* **Key Tools:**
    * `iptables` for stateful firewall configuration and NAT rules
    * `sudo` to manage administrative privileges securely
    * `ip` for network interface configuration and routing management

---

### **Configuration Guide**

#### **Step 1: Configure Network Interfaces on Debian**

The first step was to set up the Debian server as a router. I edited the /etc/network/interfaces file to assign a static IP to each network, allowing the Debian VM to serve as a default gateway for the internal Windows Server and the Kali machine. The **enp0s3** interface was configured with DHCP, which provides Internet connectivity to the router. The LAN-Windows interface (**enp0s8**) was set up as the gateway for the **192.168.60.0/24** subnet, while the LAN-Kali interface (**enp0s9**) served as the gateway for the **192.168.70.0/24** subnet.

<img width="803" height="618" alt="interfaces-conf" src="https://github.com/user-attachments/assets/94e3c3d8-af21-4f6a-b6f2-c7ccdca0d9e8" />

#### **Step 2: Enable IP Forwarding**

Now that the interfaces are configured and IPs have been designated, the next step was to enable IP forwarding to allow the Debian server to act as a router. By default, a Linux server only processes network traffic for itself. Enabling IP forwarding tells the kernel to pass traffic from one interface to the others, allowing the internal networks to communicate with the Internet.

To enable IP forwarding persistently, I edited the `/etc/sysctl.conf` file. This file contains kernel parameters that are loaded at boot time. By adding the line `net.ipv4.ip_forward = 1` to the file, the IP forwarding rule will remain active even after a system reboot.

After saving the file, the `sudo sysctl -p` command applied the changes immediately without the need to reboot the server.

#### **Step 3: Configure NAT with `iptables`**

To preserve the IP addresses of the internal systems, the Debian server must provide Network Address Translation (NAT). NAT allows internal networks to reach the Internet by translating a private IP address into a public IP address. This way, all devices on a private network can share a single public IP address, which also helps in mitigating IPv4 depletion.

To set up the NAT service, I added a `POSTROUTING` rule to the `nat` table in `iptables`. This rule specifies that any traffic leaving the router through the public-facing interface (`enp0s3`) should have its source IP address changed to the router's public IP. The `MASQUERADE` target was used because it automatically handles the router's dynamic public IP address.

Here is the rule I created:
```
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
```
To save this rule and made it persistent after reboot i saved it into the `rules.v4` file.
```
sudo iptables-save > /etc/iptables/rules.v4
```
#### **Step 4: Configure the Clients (Windows & Kali)**

The final step was to configure the client machines to use the Debian router as their gateway. For both the Windows Server and Kali Linux VMs, I manually configured a static IP address, subnet mask, and default gateway through their respective network settings GUIs.

Windows Server Configuration
The Windows Server was configured with two network adapters, each serving a specific purpose.

- Internal LAN Interface: This interface is connected to the Debian router. It uses a static IP address and serves as the Domain Controller and DNS server for the internal network.
  
    - IP Address: `192.168.60.20`
    - Subnet Mask: `255.255.255.0`
    - Default Gateway: `192.168.60.1`
    - DNS Server: `192.168.60.20` (specified as the server itself)

- Host-Only Network Interface: This interface is dedicated to sending security event logs to the Splunk server via the Splunk Universal Forwarder. Its IP address was automatically assigned via **DHCP**.

Kali Linux Configuration
The Kali Linux VM was connected to a separate LAN to simulate an external network. A static IP was assigned to ensure a consistent point of origin for simulated attacks.

- IP Address: `192.168.70.30`
- Subnet Mask: `255.255.255.0`
- Default Gateway: `192.168.70.1`
- DNS Server: `8.8.8.8` (a public DNS was used for Internet hostname resolution)"

### **Results and Verification**

* *Add screenshots of your tests here. Show the output of a successful `ping` to Google from both your Windows Server and Kali Linux to prove connectivity is working.*

---

### **Next Steps**

* Add a DHCP server to automate IP address assignment.
* Configure a local DNS server on Debian.
* Create more advanced firewall rules to filter traffic.
* Send event logs to a Splunk instance for monitoring and analysis.

---
