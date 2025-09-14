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

<img width="1319" height="897" alt="home-lab-top" src="https://github.com/user-attachments/assets/fee2793d-4774-40cd-8dd4-6bd1d6f7c862" />

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

The first step was to set up the Debian server as a multi-homed router and firewall. I edited the `/etc/network/interfaces` file to assign a static IP to each network, allowing the Debian VM to serve as the default gateway for the internal Windows Server and Kali Linux machines.

The following network interfaces were configured on the Debian router to create a secure, segmented lab environment:
- `enp0s3` (WAN Interface): This interface was configured with `DHCP` to provide Internet connectivity to the router.
- `enp0s8` (Windows LAN Interface): This interface was configured with a static IP (`192.168.60.1`) to act as the gateway for the `192.168.60.0/24` subnet.
- `enp0s9` (Kali LAN Interface): This interface was configured with a static IP (`192.168.70.1`) to act as the gateway for the `192.168.70.0/24` subnet.
- `enp0s10` (Splunk Log Interface): This interface was configured with a static IP (`192.168.56.10`) to provide a dedicated connection for log forwarding to the Splunk instance on the physical host machine.

The network configuration was then applied to the system to ensure all interfaces were active and correctly assigned.

<img width="684" height="500" alt="interfaces-conf" src="https://github.com/user-attachments/assets/a2f35e3a-018f-4581-b572-6d3851982f38" />


#### **Step 2: VirtualBox Network Configuration**

The home lab was built using the virtualization software VirtualBox to simulate a real-world network environment. The goal was to create a segmented network where an external machine (Kali Linux) attempts to interact with an internal network (Windows Server), with a Debian router/firewall filtering all traffic.

To achieve this, the following network configurations were implemented:

Debian Router/Firewall
- Adapter 1: NAT Network (for Internet access)
- Adapter 2: Internal Network (intnet-windows-firewall)
- Adapter 3: Internal Network (intnet-kali-firewall)
- Adapter 4: Host-Only Network (used for log forwarding to the Splunk instance)

Windows Server
- Adapter 1: Internal Network (intnet-windows-firewall)
- Adapter 2: Host-Only Network (used for log forwarding to the Splunk instance)

Kali Linux
- Adapter 1: Internal Network (intnet-kali-firewall)

This setup ensures that all communication between the client VMs and the Internet is routed exclusively through the Debian firewall, allowing for centralized traffic control and log analysis.

#### **Step 3: Enable IP Forwarding**

Now that the interfaces are configured and IPs have been designated, the next step was to enable IP forwarding to allow the Debian server to act as a router. By default, a Linux server only processes network traffic for itself. Enabling IP forwarding tells the kernel to pass traffic from one interface to the others, allowing the internal networks to communicate with the Internet.

To enable IP forwarding persistently, I edited the `/etc/sysctl.conf` file. This file contains kernel parameters that are loaded at boot time. By adding the line `net.ipv4.ip_forward = 1` to the file, the IP forwarding rule will remain active even after a system reboot.

After saving the file, the `sudo sysctl -p` command applied the changes immediately without the need to reboot the server.

#### **Step 4: Configure NAT with `iptables`**

To preserve the IP addresses of the internal systems, the Debian server must provide Network Address Translation (NAT). NAT allows internal networks to reach the Internet by translating a private IP address into a public IP address. This way, all devices on a private network can share a single public IP address, which also helps in mitigating IPv4 depletion.

To set up the NAT service, I added a `POSTROUTING` rule to the `nat` table in `iptables`. This rule specifies that any traffic leaving the router through the public-facing interface (`enp0s3`) should have its source IP address changed to the router's public IP. The `MASQUERADE` target was used because it automatically handles the router's dynamic public IP address.

Here is the rule I created:
```bash
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
```
To save this rule and made it persistent after reboot i saved it into the `rules.v4` file.
```bash
sudo iptables-save > /etc/iptables/rules.v4
```
#### **Step 5: Configure the Clients (Windows & Kali)**

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

#### **Step 6: Install and Configure Splunk Universal Forwarder**

To establish the centralized log collection system, the Splunk Universal Forwarder was installed on the Windows Server. This agent is designed to collect security event logs from the host and forward them to the Splunk instance for analysis.

The installation was performed using the graphical user interface (GUI) and the built-in setup wizard. During the configuration process, the forwarder was pointed to the IP address of the Splunk instance and configured to listen on the correct port to forward the Windows security event logs.

This setup ensures that security-relevant information from the Domain Controller is continuously collected, allowing for real-time monitoring and historical analysis of security events.

#### **Step 7: Configure Firewall Log Forwarding**

To ensure that all security events from the firewall are centrally monitored, the Debian router was configured to forward its logs to the Splunk instance. This is a critical step in security operations, as it allows for real-time analysis of network traffic and alerts.

To configure log forwarding, the rsyslog service, which manages system logs on Debian, was used. A new configuration file was created to instruct rsyslog to send all logs to the Splunk server.

1. Create the rsyslog Configuration File

```bash
sudo nano /etc/rsyslog.d/firewall.conf
```
2. Add the Log Forwarding Rule
The following rule was added to the new file to forward all logs to the Splunk host's IP address and listening port:

```bash
*.* @192.168.56.1:514
```

3. Restart the rsyslog Service
After saving the file, the rsyslog service was restarted to apply the changes:

```bash
sudo systemctl restart rsyslog
```

### **Results and Verification**

To verify that all network configurations were successful, I performed a series of tests from both the Windows Server and the Kali Linux machines. These tests confirmed that both clients could successfully access the Internet and that the Debian router was correctly performing NAT.

1. Internet Connectivity Test (ping to Google)
I used the ping command from both client machines to test their connectivity to an external host on the Internet. A successful ping with a low latency response indicates that the traffic is being properly forwarded and translated by the Debian router.

```Powershell
PS C:\Users\Administrator> ping google.com

Pinging google.com [142.250.178.174] with 32 bytes of data:
Reply from 142.250.178.174: bytes=32 time=19ms TTL=254
Reply from 142.250.178.174: bytes=32 time=21ms TTL=254
Reply from 142.250.178.174: bytes=32 time=19ms TTL=254
Reply from 142.250.178.174: bytes=32 time=19ms TTL=254

Ping statistics for 142.250.178.174:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 19ms, Maximum = 21ms, Average = 19ms
```

```zsh
┌──(kali㉿kali)-[~]
└─$ ping -c 4 google.com
PING google.com (142.250.184.14) 56(84) bytes of data.
64 bytes from mad41s10-in-f14.1e100.net (142.250.184.14): icmp_seq=1 ttl=254 time=19.2 ms
64 bytes from mad41s10-in-f14.1e100.net (142.250.184.14): icmp_seq=2 ttl=254 time=19.7 ms
64 bytes from mad41s10-in-f14.1e100.net (142.250.184.14): icmp_seq=3 ttl=254 time=20.0 ms
64 bytes from mad41s10-in-f14.1e100.net (142.250.184.14): icmp_seq=4 ttl=254 time=21.0 ms

--- google.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 19.194/19.962/20.995/0.658 ms
```

2. DNS Resolution Test (nslookup to Google)
I used the nslookup command to verify that the clients could resolve external hostnames. This confirms that the DNS configuration is working correctly and that the traffic can be translated and routed to the Internet.
Bash

```Powershell
PS C:\Users\Administrator> nslookup google.com
Server:  UnKnown
Address:  ::1

Non-authoritative answer:
Name:    google.com
Addresses:  2a00:1450:4003:807::200e
          142.250.178.174
```

```zsh
┌──(kali㉿kali)-[~]
└─$ nslookup google.com
Server:         8.8.8.8
Address:        8.8.8.8#53

Non-authoritative answer:
Name:   google.com
Address: 142.250.184.14
Name:   google.com
Address: 2a00:1450:4003:803::200e
```

3. Reverse DNS Lookup Test
I also performed a reverse DNS lookup to confirm that the DNS server could correctly resolve an IP address to a hostname. This is a good way to verify that your DNS configuration is working as expected.
Bash

```Powershell
PS C:\Users\Administrator> nslookup 8.8.8.8
Server:  UnKnown
Address:  ::1

Name:    dns.google
Address:  8.8.8.8
```

```zsh
┌──(kali㉿kali)-[~]
└─$ nslookup 8.8.8.8   
8.8.8.8.in-addr.arpa    name = dns.google.
```
4. Router to Splunk Connectivity Test (Port Test)

A direct ping from the Debian router to the Splunk instance was blocked by a firewall rule on the host machine. Instead, a port connectivity test was performed using nc (Netcat) to confirm that the router could successfully reach the Splunk server's listening port.

```bash
nacho@firewall-debian:~$ nc -vz 192.168.56.1 9997
192.168.56.1: inverse host lookup failed: Unknown host
(UNKNOWN) [192.168.56.1] 9997 (?) open
```

The successful results of these tests demonstrate that the Debian router is correctly configured to provide NAT, act as a gateway, and forward logs. The ping and nslookup tests confirmed that all client machines had full network connectivity to the Internet, while the port test verified that the log forwarding path from the router to the Splunk instance was also fully functional.

### **Next Steps**

* Add a DHCP server to automate IP address assignment.
* Configure a local DNS server on Debian.
* Create more advanced firewall rules to filter traffic.
* Send event logs to a Splunk instance for monitoring and analysis.

---
