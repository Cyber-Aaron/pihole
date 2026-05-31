# Network-Wide DNS Security Filter
### Deploying Pi-hole v6 in Docker on Synology NAS

![Project Status](https://img.shields.io/badge/Status-Complete-brightgreen)
![Pi-hole](https://img.shields.io/badge/Pi--hole-v6.4.2-red)
![Docker](https://img.shields.io/badge/Docker-Container-blue)
![Platform](https://img.shields.io/badge/Platform-Synology%20DS920+-orange)

---

## Overview

This project documents the deployment of Pi-hole v6 as a network-wide DNS sinkhole running inside a Docker container on a Synology DS920+ NAS. The goal was to filter malicious, unwanted, and tracking domains at the network layer — protecting all connected devices regardless of their individual security configurations — while gaining hands-on experience with Docker containerization.

> **2,472,175 domains blocked** across 4 devices on the network.

---

## Objectives

- Deploy a containerized DNS sinkhole using Docker on Synology DSM 7.3.2
- Filter known malicious, ad, and tracking domains network-wide
- Gain foundational experience with Docker and docker-compose
- Understand DNS security fundamentals in a practical home lab setting
- Document the full deployment process including troubleshooting for portfolio purposes

---

## Environment

| Component | Details |
|-----------|---------|
| **Hardware** | Synology DS920+ NAS |
| **NAS OS** | DSM 7.3.2-86009 Update 3 |
| **Container Platform** | Docker via Synology Container Manager |
| **Application** | Pi-hole v6.4.2 / Web v6.5 / FTL v6.6.2 |
| **Gateway** | Cox CGM4981COM |
| **Client Devices** | Windows Desktop, Garuda Linux, Android (Pixel 6) |

---

## Network Architecture

```
Internet
    │
Cox CGM4981COM Gateway (192.168.0.1)
    │
    ├── Synology DS920+ NAS (192.168.0.5)  ← Pi-hole DNS Server
    │       └── Docker Container: pihole
    │
    ├── Windows Desktop    (192.168.0.74)   ← DNS → 192.168.0.5
    ├── Garuda Linux       (192.168.0.237)  ← DNS → 192.168.0.5
    └── Pixel 6 Android   (192.168.0.146)  ← DNS → 192.168.0.5
```

---

## Docker Compose Configuration

```yaml
version: "3"

services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    network_mode: "host"
    environment:
      TZ: 'America/Chicago'
      WEBPASSWORD: 'YOUR_PASSWORD_HERE'
      FTLCONF_webserver_port: '8888'
      FTLCONF_ntp_active: 'false'
      FTLCONF_dns_listeningMode: 'all'
    volumes:
      - /volume1/docker/pihole/etc-pihole:/etc/pihole
      - /volume1/docker/pihole/etc-dnsmasq.d:/etc/dnsmasq.d
    restart: unless-stopped
```

### Key Configuration Decisions

**`network_mode: host`** — Configured Pi-hole to share the host network stack directly rather than using Docker's default bridge networking. This was necessary to expose real client IP addresses in the query log. Bridge mode caused all traffic to appear as originating from the Docker gateway (`172.20.0.1`), making client identification impossible.

**`FTLCONF_webserver_port: 8888`** — Pi-hole v6 defaults to port 80 for its web interface. Synology's nginx service already occupies port 80, requiring Pi-hole to be moved to an available port.

**`FTLCONF_ntp_active: false`** — Pi-hole v6 includes a built-in NTP server which conflicted with Synology's existing NTP service on port 123. Disabled to eliminate the conflict.

---

## Deployment Steps

**1. Enable SSH on Synology**
> DSM → Control Panel → Terminal & SNMP → Enable SSH

**2. Create Volume Directories**

Before deploying the container, the bind mount directories must exist on the host:
```
/volume1/docker/pihole/etc-pihole
/volume1/docker/pihole/etc-dnsmasq.d
```
Docker will not create these automatically — failure to create them before deployment results in a bind mount error.

**3. Deploy via Container Manager**
> DSM → Container Manager → Projects → Create → paste docker-compose.yml → Build

**4. Configure Blocklists**

Added the following community-maintained blocklists via Pi-hole Dashboard → Lists:

| List | Purpose |
|------|---------|
| Steven Black Hosts | Ads + malware (unified) |
| Hagezi Pro | Comprehensive multi-purpose blocking |
| Hagezi Threat Intelligence Feeds | Known malicious domains |

After adding lists, run **Tools → Update Gravity** to compile the blocking database.

**5. Configure DNS Per Device**

Due to ISP gateway limitations (Cox CGM4981COM locks DNS settings), DNS was configured manually on each device rather than at the router level.

| Device | Method |
|--------|--------|
| Windows | Network Adapter → IPv4 Properties → DNS |
| Garuda Linux | NetworkManager via `nmcli` |
| Pixel 6 | WiFi → Static IP Settings → DNS |

---

## Challenges & Troubleshooting

This section documents every significant issue encountered during deployment and how it was resolved. Real-world troubleshooting experience is central to this project's value as a learning exercise.

---

### 1. Bind Mount Failure on Container Start
**Error:** `Bind mount failed: '/docker/pihole/etc-pihole' does not exist. Exit code: 1`

**Root Cause:** Docker requires bind mount directories to exist on the host before container startup. They are not created automatically.

**Resolution:** Created the required directory structure in Synology File Station before deployment. Also corrected the volume paths to include `/volume1/` prefix, which is required on Synology systems.

---

### 2. Pi-hole Web UI Password Not Working
**Symptom:** Login to the Pi-hole dashboard failed despite copying the password directly from the docker-compose.yml `WEBPASSWORD` environment variable.

**Root Cause:** Pi-hole v6 changed the password handling behavior. The `WEBPASSWORD` environment variable does not reliably set the password on first run in v6.

**Resolution:** Reset the password directly inside the running container:
```bash
sudo docker exec -it pihole pihole setpassword yournewpassword
```
Note: Pi-hole v5 used `pihole -a -p` for this command. v6 changed the syntax to `pihole setpassword`.

---

### 3. All Clients Showing as 172.20.0.1
**Symptom:** Pi-hole query log showed all DNS queries originating from `172.20.0.1` regardless of which device made the request.

**Root Cause:** Docker's default bridge networking mode performs NAT (Network Address Translation) on traffic passing through the container. All client IPs were replaced with the Docker bridge gateway address, making individual device identification impossible.

**Resolution:** Switched to `network_mode: host` in docker-compose.yml. This removes Docker's network isolation and allows Pi-hole to see real client IP addresses directly.

---

### 4. Web UI Inaccessible After Switching to Host Networking
**Symptom:** Pi-hole dashboard became unreachable after adding `network_mode: host`.

**Root Cause:** In bridge mode, port mapping (`8080:80`) forwarded external port 8080 to Pi-hole's internal port 80, avoiding the conflict with Synology's nginx. In host mode, port mapping is not used — Pi-hole directly occupies port 80, which was already in use by Synology's nginx web server.

**Resolution:** Used the Pi-hole v6 environment variable `FTLCONF_webserver_port: '8888'` to move Pi-hole's web interface to an available port.

---

### 5. IPv6 DNS Servers Bypassing Pi-hole
**Symptom:** Despite configuring `192.168.0.5` as the DNS server on devices, some traffic was not appearing in the Pi-hole query log.

**Root Cause:** The Cox gateway automatically pushes IPv6 DNS servers (`2001:578:3f::30`) to connected devices via DHCPv6. Modern operating systems and devices prefer IPv6 DNS over IPv4 when both are available, causing DNS queries to bypass Pi-hole entirely.

**Resolution:**
- **Windows:** Disabled IPv6 on the network adapter via adapter properties
- **Garuda Linux:** Used NetworkManager to ignore auto DNS and disable IPv6 DNS:
```bash
nmcli connection modify "NetworkName" ipv6.ignore-auto-dns yes
nmcli connection modify "NetworkName" ipv6.method "disabled"
```
- **Pixel 6:** Disabled Private DNS (Settings → Network & Internet → Private DNS → Off)

---

### 6. Phone on Wrong Subnet (192.168.1.x vs 192.168.0.x)
**Symptom:** Pixel 6 received IP address `192.168.1.128` with gateway `192.168.1.1`, placing it on a completely different subnet from the NAS and other devices. The phone could not reach Pi-hole at `192.168.0.5`.

**Root Cause:** During earlier troubleshooting, a static IP was manually configured on the phone using incorrect subnet values. The Cox gateway assigns wired devices to `192.168.0.x` and WiFi devices dynamically — the manual static configuration placed the phone on the wrong subnet.

**Resolution:** Reset the phone's IP configuration back to DHCP, allowing it to obtain a correct `192.168.0.x` address from the router, then manually set only the DNS field to `192.168.0.5`.

---

### 7. Pi-hole v6 Configuration Variable Changes
**Symptom:** Several environment variables documented in online guides and Pi-hole's own documentation did not function as expected.

**Root Cause:** Pi-hole v6 was a major architectural rewrite released in early 2025. Many configuration variables changed to use a `FTLCONF_` prefix. The majority of available documentation online was written for v5 and did not reflect v6 changes.

**Resolution:** Referenced Pi-hole v6 release notes and tested variables iteratively. Key v6 changes documented:

| Function | v5 Variable | v6 Variable |
|----------|------------|-------------|
| Web server port | `WEB_PORT` | `FTLCONF_webserver_port` |
| NTP server | N/A | `FTLCONF_ntp_active` |
| DNS listening mode | `DNSMASQ_LISTENING` | `FTLCONF_dns_listeningMode` |
| Password reset | `pihole -a -p` | `pihole setpassword` |

---

## Results

| Metric | Value |
|--------|-------|
| **Total domains blocked** | 2,472,175 |
| **Blocklists active** | 7 |
| **Devices protected** | 4 |
| **Container uptime** | Continuous (restart: unless-stopped) |
| **DNS queries handled** | All IPv4 traffic on configured devices |

---

## Security Concepts Demonstrated

**DNS Sinkhole** — Pi-hole intercepts DNS queries for known malicious or unwanted domains and returns a null response, preventing devices from connecting to those destinations entirely.

**Defense in Depth** — This deployment adds a network-layer security control independent of endpoint security software. Even devices without antivirus or ad blockers benefit from Pi-hole's filtering.

**Network Segmentation Awareness** — During deployment, a subnet isolation issue was identified where the ISP gateway was separating wired and wireless devices onto different network segments, preventing communication between them.

**Container Security** — Docker containerization isolates Pi-hole from the host OS, limiting the blast radius of any potential compromise and simplifying updates and rollbacks via container rebuilds.

**Traffic Visibility** — The Pi-hole query log provides full visibility into DNS requests across all network devices, enabling detection of unusual behavior such as unexpected external connections or potential C2 (command and control) communication attempts.

**Threat Intelligence Integration** — Community-maintained blocklists aggregate threat intelligence from multiple sources, providing up-to-date protection against known malicious infrastructure.

---

## Lessons Learned

- **Version compatibility matters** — Pi-hole v6 introduced breaking changes to configuration variables and CLI commands. Always verify documentation matches the deployed version before troubleshooting.
- **ISP gateways limit control** — Consumer ISP-provided hardware significantly restricts network configuration options. A dedicated router behind the ISP gateway would provide full DNS control at the network level.
- **IPv6 is often overlooked** — IPv6 DNS bypass is a common blind spot in home network security configurations. Any network security control applied only to IPv4 may be trivially bypassed.
- **Docker networking modes have security implications** — The choice between bridge and host networking affects not just connectivity but security visibility and isolation trade-offs.
- **Document as you go** — Capturing errors and resolutions in real time produces far more accurate and useful documentation than reconstructing from memory afterward.

---

## Future Improvements

- [ ] Deploy own router behind ISP gateway (bridge mode) for full network-level DNS control
- [ ] Implement DNS over HTTPS (DoH) on Pi-hole using Cloudflared for encrypted upstream queries
- [ ] Deploy WireGuard VPN on Synology NAS to route mobile traffic through Pi-hole when off network
- [ ] Configure Unbound as a recursive DNS resolver to eliminate reliance on upstream DNS providers
- [ ] Set up monitoring and alerting for unusual DNS query patterns
- [ ] Explore Pi-hole DHCP server as replacement for ISP gateway DHCP

---

## References

- [Pi-hole Official Documentation](https://docs.pi-hole.net)
- [Pi-hole v6 Release Notes](https://pi-hole.net)
- [Pi-hole Docker Hub](https://hub.docker.com/r/pihole/pihole)
- [Synology Container Manager Documentation](https://www.synology.com)
- [Hagezi DNS Blocklists](https://github.com/hagezi/dns-blocklists)
- [Steven Black Hosts](https://github.com/StevenBlack/hosts)

---

## Author

**Kenneth** — Aspiring Cybersecurity Professional  
Currently pursuing: Google Cybersecurity Professional Certificate  
Next certifications: CompTIA Security+, Linux+, Network+  

*This project is part of an ongoing home lab portfolio documenting hands-on cybersecurity and IT skills.*

---

> *"The best way to learn security is to break things safely, fix them, and document why."*
