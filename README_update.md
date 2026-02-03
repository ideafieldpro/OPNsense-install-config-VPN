# **OPNsense Firewall: Production Deployment Case Study (DHCP + VPN)**

## **Project Overview**

This case study documents a production-style deployment of the OPNsense firewall on bare-metal hardware, acting as the secure entry point to a private infrastructure. The appliance terminates WAN traffic, provides DHCP services for the LAN, and exposes a WireGuard-based VPN for remote administration of internal servers. The focus is on designing a clean network baseline, making opinionated configuration choices, and validating that the environment is maintainable and observable over time. [OPNsense Documentation](https://docs.opnsense.org/manual/install.html)

## **Case Study Summary**

- **Scenario**: Deploy a dedicated OPNsense firewall to front a small production network, consolidating perimeter security, DHCP, and VPN access into a single hardened appliance.
- **Constraints**: Bare-metal hardware with dual NICs, low idle power usage, and a simple /24 LAN segment to support servers, management devices, and end-user clients.
- **Approach**: Use OPNsense with ZFS, Intel NICs, Dnsmasq for integrated DHCP, and WireGuard for modern, low-overhead VPN access. Document all steps and decisions as an operator runbook.
- **Outcome**: A reproducible configuration that delivers stable LAN connectivity, controlled static mappings for critical hosts, and secure remote access into the LAN over VPN, backed by regular backups and monitoring.

## **Architecture Overview**

At a high level, the deployment places OPNsense between the ISP-provided WAN and an internal LAN, with DHCP-managed clients on the LAN and WireGuard VPN clients joining via a dedicated tunnel subnet.

```mermaid
flowchart LR
  wan[\"WAN / Internet\"]
  opnsense[\"OPNsense Firewall\"]
  lan[\"LAN Switch & Clients (192.168.1.0/24)\"]
  vpnClients[\"WireGuard VPN Clients (10.10.10.0/24)\"]

  wan --> opnsense
  vpnClients --- opnsense
  opnsense --> lan
```

Key design elements:

- **WAN**: DHCP or static from ISP, terminated on OPNsense WAN interface.
- **LAN**: `192.168.1.0/24` with OPNsense at `192.168.1.1`, serving as default gateway and DNS for clients.
- **DHCP**: Dynamic range `192.168.1.100–192.168.1.200`, with static mappings reserved below `.100` for servers and infrastructure.
- **VPN**: WireGuard tunnel network `10.10.10.0/24`, with clients allowed to reach select LAN subnets.

## **Skills Demonstrated**

✓ Network infrastructure design and implementation (WAN/LAN segmentation, DHCP ranges, static mappings)  
✓ Firewall configuration and security hardening for a production-style perimeter device  
✓ DHCP server management and IP allocation using Dnsmasq on OPNsense 25.x  
✓ VPN deployment for secure remote access (WireGuard road-warrior setup into LAN resources)  
✓ Production environment best practices (backups, updates, monitoring, logging)  
✓ Technical documentation and knowledge transfer through a step-by-step operator runbook  
✓ Troubleshooting and problem resolution for connectivity, DHCP, and VPN issues  

## **Objectives**

- Install OPNsense on dedicated hardware for optimal network security
- Configure dual-NIC setup for WAN/LAN traffic routing
- Implement DHCP services for automated IP management
- Establish VPN access for secure remote server administration
- Document production-ready firewall deployment

***

## **Table of Contents**

1. [Hardware Requirements](#hardware-requirements)
2. [Pre-Installation Preparation](#pre-installation-preparation)
3. [Installation Process](#installation-process)
4. [Initial Configuration](#initial-configuration)
5. [DHCP Configuration](#dhcp-configuration-dnsmasq)
6. [VPN Setup](#vpn-setup)
7. [Firewall Rules](#firewall-rules)
8. [Troubleshooting](#troubleshooting)
9. [Best Practices](#best-practices)

***

## **Technical Implementation**

### **Hardware Requirements**

### **Minimum Specifications** (per [OPNsense documentation](https://docs.opnsense.org/manual/hardware.html)) [docs.opnsense](https://docs.opnsense.org/manual/hardware.html)

- **CPU**: 1.5 GHz dual-core processor (Intel/AMD x86-64)
- **RAM**: 4 GB minimum (8 GB recommended for production)
- **Storage**: 40 GB SSD (for optimal performance)
- **Network Interfaces**: Minimum 2 NICs (WAN + LAN)
  - Intel NICs strongly recommended for stability
  - Realtek NICs may have driver limitations

### **Recommended Production Hardware**

For production environments with VPN and DHCP services: [reddit](https://www.reddit.com/r/opnsense/comments/1gpvb8h/what_hardware_requirements_for_bare_metal/)

- **CPU**: Intel i5 or equivalent (T-series for lower power consumption)
- **RAM**: 16 GB DDR4
- **Storage**: 120 GB SSD (enterprise-grade)
- **NICs**: Dual Intel Gigabit or 2.5GbE interfaces
- **Power**: ~12-50W idle consumption depending on hardware

***

### **Pre-Installation Preparation**

### **1. Download OPNsense ISO**

Visit official download page: [https://opnsense.org/download/](https://opnsense.org/download/)

Verify checksum after download

```bash
sha256sum OPNsense-XX.X-OpenSSL-dvd-amd64.iso
```

Compare with checksum listed on download page

### **2. Create Bootable USB**

**Linux/macOS:**

```bash
dd if=OPNsense-XX.X-OpenSSL-dvd-amd64.iso of=/dev/sdX bs=4M status=progress
sync
```

**Windows:**

- Use [Rufus](https://rufus.ie/) or [balenaEtcher](https://www.balena.io/etcher/)
- Select ISO file and target USB drive
- Write in DD Image mode

### **3. Network Planning**

Document your network topology before installation:

| Interface | Purpose              | Network Range        | Gateway        |
|-----------|----------------------|----------------------|----------------|
| WAN       | Internet Connection  | DHCP/Static from ISP | ISP Gateway    |
| LAN       | Internal Network     | 192.168.1.0/24       | 192.168.1.1    |

***

### **Installation Process**

This section covers installing OPNsense onto bare-metal hardware using bootable media. The goal is to end up with a resilient base system (preferably using ZFS) that can support firewalling, DHCP, and VPN services without manual intervention after reboots.

### **Step 1: Boot from Installation Media**

1. Insert USB drive into target hardware
2. Access BIOS/UEFI (typically F2, F12, or DEL)
3. Set boot priority to USB device
4. Save and reboot

### **Step 2: OPNsense Installer**

1. **Login Credentials** (at boot screen): [docs.opnsense](https://docs.opnsense.org/manual/install.html)
   - Username: `installer`
   - Password: `opnsense`

2. **Select Installation Options**:
   - Choose **"Install (UFS)"** or **"Install (ZFS)"**
   - ZFS recommended for data integrity and snapshots

3. **Disk Configuration**:
   - Select target installation disk
   - For ZFS: Choose **"stripe"** for single disk setups

4. **Set Root Password**:
   - Create strong password for root account
   - **Document this securely** - required for all administrative access

5. **Complete Installation**:
   - Wait for installation to complete (~5-10 minutes)
   - Remove installation media when prompted
   - System will reboot

### **Step 3: Interface Assignment**

Upon first boot, you'll see the console menu: [docs.opnsense](https://docs.opnsense.org/manual/install.html)

```bash
0) Logout                    7) Ping host
1) Assign interfaces         8) Shell
2) Set interface(s) IP       9) pfTop
3) Reset root password      10) Filter logs
4) Reset to factory         11) Restart web interface
5) Reboot system            12) Upgrade from console
6) Halt system              13) Restore configuration
```

**If interfaces aren't auto-detected:**

1. Select **Option 1** - Assign interfaces
2. Enter interface names when prompted:
   - **WAN**: Typically `vnet0`, `em0`, `igb0`, or `re0`
   - **LAN**: Typically `vnet1`, `em1`, `igb1`, or `re1`
3. Confirm assignments with `y`

**Tip**: Identify NICs by MAC addresses or temporarily connect to known networks

***

### **Initial Configuration**

Here you bring the newly installed firewall onto the network by assigning IPs and enabling basic services. The objective is to establish a secure management plane on the LAN, with HTTPS access to the web GUI and a small initial DHCP scope for clients.

### **Step 1: Configure LAN Interface**

1. Select **Option 2** - Set interface(s) IP address
2. Choose **LAN** interface (typically option 2)
3. Configure IPv4:

   ```bash
   IPv4 Address: 192.168.1.1
   Subnet Mask: 24 (255.255.255.0)
   ```

4. Skip IPv6 if not needed
5. **Enable DHCP server**: `y`
   - Start address: `192.168.1.100`
   - End address: `192.168.1.200`
6. **Do NOT revert to HTTP**: `n` (keep HTTPS)

### **Step 2: Access Web GUI**

1. Connect client computer to LAN interface
2. Open browser and navigate to: `https://192.168.1.1`
3. Accept self-signed certificate warning
4. **Login credentials**:
   - Username: `root`
   - Password: [your configured password]

### **Step 3: Initial Setup Wizard**

The web interface will launch a configuration wizard:

1. **General Information**:
   - Hostname: `firewall`
   - Domain: `yourdomain.local`
   - Primary DNS: `8.8.8.8` or `1.1.1.1`
   - Secondary DNS: `8.8.4.4` or `1.0.0.1`

2. **Time Server**:
   - NTP Server: `pool.ntp.org`
   - Timezone: Select your timezone

3. **WAN Configuration**:
   - Type: DHCP (for most ISP connections) or Static IP
   - If static, enter ISP-provided details

4. **Set Admin Password**:
   - Change default root password if not done during install

5. **Reload Configuration**

***

***

### **DHCP Configuration (Dnsmasq)**

This section configures DHCP using the modern Dnsmasq service in OPNsense 25.x, replacing the legacy ISC DHCP daemon. The intent is to centralize IP address management, keep static infrastructure addresses predictable, and integrate DHCP with local DNS so hosts are easily discoverable.

### **Enable Dnsmasq Services**

Navigate to: **Services → Dnsmasq DNS & DHCP → Settings** [docs.opnsense](https://docs.opnsense.org/manual/dnsmasq.html)

#### **General Configuration:**

1. **Enable Dnsmasq**: ☑ Enable
2. **Interfaces**: Select specific interfaces (e.g., LAN)
   - **Do not leave on "All"** - restrict to intended DHCP interfaces only
3. **Listen Port**:
   - `53` (for DNS functionality)
   - Set to `0` to disable DNS if using Unbound
4. **DNSSEC**: ☑ Enable (if desired for DNS validation)
5. **Local Domain**: `yourdomain.local`
6. **Register DHCP Leases**: ☑ Enable (allows hostname resolution for DHCP clients) [reddit](https://www.reddit.com/r/opnsense/comments/1nx4bec/migrated_from_isc_to_dnsmasq_dhcp_steps_thoughts/)
7. **Register Static Mappings**: ☑ Enable
8. Click **Save**

***

### **Configure DHCP Ranges**

Navigate to: **Services → Dnsmasq DNS & DHCP → DHCP** [youtube](https://www.youtube.com/watch?v=AzJB6Mx3CnQ&vl=te)

#### **Add IPv4 DHCP Range:**

1. Click **+** to add new range
2. **Interface**: LAN
3. **Network**: 192.168.1.0/24
4. **Gateway**: 192.168.1.1
5. **Range Configuration**:
   - **Start**: 192.168.1.100
   - **End**: 192.168.1.200
6. **Lease Time**: 
   - **Default**: 12h (43200 seconds)
   - **Max**: 24h (86400 seconds)
7. **DNS Servers** (optional): Leave blank to use OPNsense IP automatically
8. **Domain**: yourdomain.local
9. Click **Save**

**Important**: The default behavior assigns OPNsense's LAN IP as both gateway and DNS server automatically. [docs.opnsense](https://docs.opnsense.org/manual/dnsmasq.html)

***

### **Configure DHCP Options** (Advanced)

Navigate to: **Services → Dnsmasq DNS & DHCP → DHCP Options** [danb](https://danb.me/blog/opnsense-dns/)

For custom DNS servers or advanced configurations:

1. Click **+** to add new option
2. **Option**: `dns-server`  [youtube](https://www.youtube.com/watch?v=AzJB6Mx3CnQ)
3. **Value**: `192.168.1.1` or custom DNS (e.g., `1.1.1.1,8.8.8.8`)
4. **Interface**: LAN (or leave blank for all interfaces)
5. **Description**: Custom DNS servers
6. Click **Save** and **Apply**

**Common DHCP Options**: [docs.opnsense](https://docs.opnsense.org/manual/dnsmasq.html)

| Option      | Code | Purpose           | Example Value       |
| ----------- | ---- | ----------------- | ------------------- |
| router      | 3    | Default gateway   | 192.168.1.1         |
| dns-server  | 6    | DNS servers       | 192.168.1.1         |
| domain-name | 15   | Local domain      | yourdomain.local    |
| ntp-server  | 42   | Time server       | 192.168.1.1         |

***

### **Static DHCP Reservations** (Servers/Critical Devices)

Navigate to: **Services → Dnsmasq DNS & DHCP → Static Mappings** [youtube](https://www.youtube.com/watch?v=AzJB6Mx3CnQ)

#### **Add Static Mapping:**

1. Click **+** to add new static reservation
2. Configure:
   - **Interface**: LAN
   - **MAC Address**: 00:11:22:33:44:55
   - **IP Address**: 192.168.1.10
   - **Hostname**: server01
   - **Description**: Production Application Server
   - **DNS Domain**: yourdomain.local (optional)
3. Click **Save** and **Apply**

**Example Static Reservations**:

| Device         | MAC Address       | Static IP    | Hostname   | Description                |
| -------------- | ----------------- | ------------ | ---------- | -------------------------- |
| Server01       | 00:11:22:33:44:55 | 192.168.1.10 | server01   | Primary Application Server |
| NAS            | 00:11:22:33:44:66 | 192.168.1.11 | nas01      | Network Storage            |
| Printer        | 00:11:22:33:44:77 | 192.168.1.12 | printer01  | Office Printer             |
| Mgmt Interface | 00:11:22:33:44:88 | 192.168.1.5  | mgmt-switch| Core Switch Management     |

**Best Practice**: Reserve IPs outside your DHCP range (e.g., .1-.99 for static, .100-.200 for DHCP pool). [youtube](https://www.youtube.com/watch?v=AzJB6Mx3CnQ&vl=te)

***

### **DNS Integration with Unbound**

For local hostname resolution using Unbound DNS: [reddit](https://www.reddit.com/r/opnsense/comments/1kusje3/guide_to_configuremigrate_to_dnsmasq_forward_dns/)

1. Navigate to **Services → Unbound DNS → General**
2. **Enable Unbound**: ☑ Enable
3. **Network Interfaces**: Select LAN
4. **DHCP Registration**:
   - Navigate to **Services → Unbound DNS → Advanced**
   - ☑ **Register DHCP leases**: Enables Dnsmasq to forward DHCP hostname registrations to Unbound
   - ☑ **Register DHCP static mappings**
5. **Forward Local Domain** (if using Dnsmasq for DNS):
   - Navigate to **Services → Unbound DNS → Query Forwarding**
   - Click **+** to add override
   - **Domain**: yourdomain.local
   - **Server IP**: 127.0.0.1@5353 (forward to Dnsmasq)
6. Click **Save** and **Apply**

This configuration allows Unbound to handle external DNS queries while Dnsmasq manages local DHCP hostname registration. [reddit](https://www.reddit.com/r/opnsense/comments/1nx4bec/migrated_from_isc_to_dnsmasq_dhcp_steps_thoughts/)

***

### **Migrating from ISC DHCP** (If Upgrading)

If migrating from ISC DHCP to Dnsmasq: [youtube](https://www.youtube.com/watch?v=fsbMvI7beeA)

1. **Export Static Mappings**:
   - Navigate to **Services → ISC DHCPv4 → [Interface]**
   - Click **Export** to download CSV of static mappings

2. **Configure Dnsmasq** (as described above)

3. **Import Static Mappings**:
   - Navigate to **Services → Dnsmasq DNS & DHCP → Static Mappings**
   - Click **Import**
   - Upload previously exported CSV

4. **Disable Router Advertisements** (if not using IPv6):
   - Navigate to **Services → Router Advertisements**
   - Uncheck all interfaces
   - Click **Save**

5. **Switch Services**:
   - Navigate to **Services → ISC DHCPv4**
   - ☐ Disable all interfaces
   - Navigate to **Services → Dnsmasq DNS & DHCP → Settings**
   - ☑ Enable Dnsmasq
   - Click **Apply**

6. **Verify Functionality**:
   - Check **Services → Dnsmasq DNS & DHCP → Leases** for active assignments
   - Test client DHCP renewal

***

### **Verify DHCP Operation**

Check active leases: **Services → Dnsmasq DNS & DHCP → Leases** [docs.opnsense](https://docs.opnsense.org/manual/dnsmasq.html)

You should see:

- **IP Address**: Assigned client IPs
- **MAC Address**: Client hardware addresses
- **Hostname**: Client device names
- **Lease Start/End**: Lease validity period

**Troubleshooting**:

- Ensure Dnsmasq service is running: **Services → Dnsmasq DNS & DHCP → Settings** (check enabled)
- Verify interface selection matches your LAN interface
- Check firewall rules allow DHCP traffic (UDP ports 67/68)
- Review logs: **Services → Dnsmasq DNS & DHCP → Log File**

***

## **VPN Setup**

### **WireGuard VPN Configuration** (Recommended)

WireGuard is used here as the primary remote access VPN because it is lightweight, easy to manage with key pairs, and performs well on modest hardware. The design follows a \"road-warrior\" pattern where individual clients receive a single /32 address in a dedicated VPN subnet and are allowed to access selected LAN networks.

Navigate to: **VPN → WireGuard**

#### **Step 1: Enable WireGuard**

1. Go to **VPN → WireGuard → Settings**
2. ☑ Enable WireGuard
3. Click **Save**

#### **Step 2: Create Local Server**

1. Navigate to **Local** tab
2. Click **Add**
3. Configure:
   - **Name**: wg0
   - **Listen Port**: 51820
   - **Tunnel Address**: 10.10.10.1/24
   - **Peers**: (configure after creating peers)
4. Click **Save**

#### **Step 3: Create Peer Configuration**

1. Navigate to **Peers** tab
2. Click **Add**
3. Configure:
   - **Name**: admin-laptop
   - **Public Key**: [generated from client]
   - **Allowed IPs**: 10.10.10.2/32
   - **Endpoint Address**: [leave empty for road warrior]
4. Click **Save**

#### **Step 4: Firewall Rules for VPN**

1. Navigate to **Firewall → Rules → WAN**
2. Click **Add** (top arrow for prepend)
3. Configure:
   - **Action**: Pass
   - **Interface**: WAN
   - **Protocol**: UDP
   - **Destination Port**: 51820
   - **Description**: Allow WireGuard VPN
4. **Save** and **Apply Changes**

5. Navigate to **Firewall → Rules → WireGuard**
6. Click **Add**
7. Configure:
   - **Action**: Pass
   - **Interface**: WireGuard
   - **Source**: any
   - **Destination**: LAN net
   - **Description**: Allow VPN to LAN
8. **Save** and **Apply Changes**

#### **Step 5: Client Configuration**

Find or create the client configuration file on your client device at `/etc/wireguard/wg0.conf` or use a WireGuard app to generate keys.

Generate client configuration file:

```ini
[Interface]
PrivateKey = [CLIENT_PRIVATE_KEY]
Address = 10.10.10.2/32
DNS = 192.168.1.1

[Peer]
PublicKey = [SERVER_PUBLIC_KEY]
Endpoint = [YOUR_PUBLIC_IP]:51820
AllowedIPs = 192.168.1.0/24
PersistentKeepalive = 25
```

***

## **Firewall Rules**

### **Essential LAN Rules**

Firewall rules control how LAN devices, VPN clients, and the firewall itself can communicate. The aim is to start with a simple, auditable baseline (LAN-to-any) and then tighten access over time using the principle of least privilege.

Navigate to: **Firewall → Rules → LAN**

#### **Default LAN to Any** (usually pre-configured):

- **Action**: Pass
- **Interface**: LAN
- **Source**: LAN net
- **Destination**: any
- **Description**: Default allow LAN to any

### **Best Practices**:

1. **Principle of Least Privilege**: Only allow necessary traffic
2. **Log Important Rules**: Enable logging for security analysis
3. **Regular Review**: Audit rules quarterly
4. **Document Changes**: Add meaningful descriptions

***

### **Troubleshooting**

### **Cannot Access Web Interface**

1. Verify LAN IP configuration:
   - Console: Option 2 → View current IP
2. Check client network settings:
   - Must be on same subnet (192.168.1.x)
   - Gateway should point to OPNsense LAN IP
3. Clear browser cache/try incognito mode
4. Disable client firewall temporarily

### **No Internet Access from LAN**

1. Verify WAN interface has IP address
   - Console → Option 2 → Select WAN → View
2. Check gateway configuration:
   - **System → Gateways → Single**
3. Verify DNS settings:
   - **System → Settings → General**
4. Test from OPNsense:
   - Console → Option 7 → Ping 8.8.8.8

### **DHCP Not Assigning Addresses**

1. Verify DHCP service is running:
   - **Services → Dnsmasq DNS & DHCP → Settings** → Ensure **Enable Dnsmasq** is checked
2. Restart DHCP service:
   - **Services → Dnsmasq DNS & DHCP → Settings** → Click **Apply** to restart
3. Check DHCP leases:
   - **Services → Dnsmasq DNS & DHCP → Leases**
4. Verify firewall rules allow DHCP (UDP ports 67/68) on the LAN interface

***

### **Best Practices**

### **1. Regular Backups** [youtube](https://www.youtube.com/watch?v=UI5tO1hP2q8)

- Navigate to **System → Configuration → Backups**
- Download configuration XML weekly/monthly
- Store securely off-site

### **2. Firmware Updates**

- Check: **System → Firmware → Updates**
- Enable automatic security updates
- Review changelog before major updates
- Schedule updates during maintenance windows

### **3. Security Hardening**

- Change default SSH port
- Enable two-factor authentication
- Implement firewall rule logging
- Regular log review via **System → Log Files**
- Consider IDS/IPS (Suricata) for advanced threat detection

### **4. Monitoring**

- Enable dashboard widgets for at-a-glance status
- Monitor DHCP lease utilization
- Track VPN connection logs
- Set up email alerts for critical events

### **5. Documentation**

- Maintain network diagram
- Document all static mappings
- Keep firewall rule inventory
- Record configuration changes

***

## **Additional Resources**

### **Official Documentation**

- [OPNsense Official Documentation](https://docs.opnsense.org/)
- [OPNsense Hardware Guide](https://docs.opnsense.org/manual/hardware.html)

### **Community Guides & Videos**

- [OPNsense Forum](https://forum.opnsense.org/)
- DHCP and DNSmasq deep dive: [How to set up DHCP server on OPNsense (Zenarmor)](https://www.zenarmor.com/docs/network-security-tutorials/how-to-setup-dhcp-server-on-opnsense)


***

**Author**: Craig Sheffield (ideafieldpro)  