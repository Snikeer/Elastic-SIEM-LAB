# Elastic SIEM Lab

I built this lab to learn Elastic Security hands-on. I have studied SIEM concepts during my cyber security bachelor and in the Google Cybersecurity Professional Certificate, but I had never worked with the Elastic before. The goal here is to get comfortable with how Elastic does it: how logs get from a host into Elasticsearch, how Fleet manages the agents, and how detection rules are written and tested in Kibana. Everything runs inside VMware.

The setup is small: one Elastic Stack server, one Linux host that gets monitored, one Kali VM that attacks it, and the detection rules I write to catch those attacks.

This repository documents what I did, why I did it, and what went wrong along the way. I kept the commands and the screenshots so the results can be checked.

## Why I built it

I wanted hands-on experience with Elastic Security, because many job postings name it and reading the documentation is not the same as running the stack. I also wanted to see how a log gets from a host into Elasticsearch, and why a detection can fail even when the rule looks correct in Kibana. Network segmentation came later, as a second track. I wanted Elastic working before I started changing firewall rules, so I would know which part caused a problem.

## Tools and versions

- Windows host running VMware Workstation Pro.
- Ubuntu Server 24.04 LTS for the Elastic server and the monitored host.
- Kali Linux VM as the attacker.
- Elastic Stack 9.5.3 (Elasticsearch, Kibana, Fleet Server, Elastic Agent), the version that was current when I started.

The host has 100 GB of RAM, which is far more than this lab needs. The VM sizes below are what I chose, not the minimum. I have not tried smaller sizes, so I cannot say how low they can go.

## Terms used in this README

**SIEM (Security Information and Event Management):** a system that collects logs from many machines, stores them in one place, and runs rules that flag suspicious activity.

**Elasticsearch:** the search and storage engine that holds the logs.

**Kibana:** the web interface where I search logs, view dashboards, and manage detection rules.

**Fleet Server:** the component that manages Elastic Agents from one central place.

**Elastic Agent:** a small program installed on each monitored host. It collects logs and sends them to Elasticsearch through Fleet.

**Integration:** a ready-made package that tells the agent which logs to collect and how to read them.

**Data stream:** how Elasticsearch stores time-based logs, so they can be searched and kept according to a retention policy.

**Detection rule:** a condition that runs over incoming events and creates an alert when it matches.

**ECS (Elastic Common Schema):** the standard field names Elastic uses, for example `source.ip` or `user.name`. Rules are easier to write when every log source uses the same names.

**MITRE ATT&CK:** a public catalogue of attacker techniques. Each detection in this lab is linked to one or more of them.

## Lab design

### Why two tracks

The first track is the SIEM on a flat network. I wanted the SIEM to work before I added firewall rules, so that a broken log pipeline could only be caused by the SIEM setup. The second track is segmentation, which I added after logs were already flowing. That way each firewall rule could be tested against real data.

### Track 1: flat lab network

| VM | IP on lab-net | RAM | vCPU | Disk | Role |
|---|---|---|---|---|---|
| Windows host | 192.168.100.1 | (host) | (host) | (host) | Browser for Kibana, SSH client to make my life easyer |
| `elk-siem-node` | 192.168.100.10 | 16 GB | 4 | 80 GB | Elasticsearch, Kibana, Fleet Server |
| `target-node` | 192.168.100.20 | 4 GB | 2 | 30 GB | Monitored host, DVWA in Docker, Elastic Agent |
| `kali` | 192.168.100.30 | 8 GB | 2 | existing | Attacker |

All lab VMs sit on one host-only network called `lab-net`, `192.168.100.0/24`. The Windows host has `.1` on this network, so I can open Kibana in the browser and SSH into each VM from PowerShell or VS Code without switching windows. Keeping the lab reachable from the host was a practical choice for working faster, not a security decision.

Each VM also has a second adapter on NAT. I use it only for package updates. Attack traffic never goes through it, so it does not change what the lab tests.

### Track 2: segmentation (chapter 11)

Companies rarely run everything on one network, and firewall logs are among the most useful sources a security team has. In this track I add a fourth VM, `fw-node`, as a router and firewall running nftables. It splits the lab into four zones with default deny between them, and its logs go to the same SIEM.

| Zone | Network | Hosts |
|---|---|---|
| mgmt | 10.10.10.0/24 | Windows host, fw-node |
| attack | 10.10.20.0/24 | kali |
| dmz | 10.10.30.0/24 | target-node |
| security | 10.10.40.0/24 | elk-siem-node |

The rules I plan to allow are these. The Windows host reaches every zone over SSH, and Kibana in the security zone. The attack zone reaches the dmz zone, since that is where the attacks land. The dmz zone reaches Fleet and Elasticsearch in the security zone, on those ports only, because agents start the connection. The attack zone has no path to the security zone. I want to confirm that with a test from Kali, not just assume it.

## Phases

- [X] Phase 0: Design
- [X] Phase 1: Virtual machines and flat network
- [ ] Phase 2: Base hardening
- [ ] Phase 3: Elasticsearch and Kibana
- [ ] Phase 4: Fleet Server and Elastic Agent
- [ ] Phase 5: Baseline data collection
- [ ] Phase 6: DVWA target
- [ ] Phase 7: Attack simulation from Kali
- [ ] Phase 8: Detection rules, tested against the attacks
- [ ] Phase 9: Dashboard and alerting
- [ ] Phase 10: Summary and lessons learned
- [ ] Phase 11: Segmentation with fw-node

## Implementation log

Each phase below follows the same pattern. First what I did, then why, then the commands, then the output I got, and last the problems I ran into and how I solved them. Screenshots only appear where they show something I want to prove.


### Phase 1: Lab network and virtual machines

#### Background
Phase 0 set the plan: a flat network first, so the SIEM can be checked before any firewall rules are added. This phase builds the network and the three VMs the rest of the lab depends on. Nothing from Elastic is installed yet.


#### 1. Choosing Ubuntu Server 24.04
Before creating any VM I checked the Elastic support matrix for version 9.5.3. A wrong OS choice will be hard to fix after, so this was the first thing I looked at. Ubuntu 24.04 is listed as supported.
https://www.elastic.co/support/matrix 

<img src="docs/screenshots/00_elastic_support_matrix.png" alt="Elastic support matrix for Elastic Stack 9.5.3 showing Ubuntu 24.04" width="700">


#### 2. Creating the lab network
In VMware Workstation I opened **Edit > Virtual Network Editor** and clicked **Change Settings**, because the editor needs administrator rights. I selected VMnet2 and set it to Host-only. I unticked "Use local DHCP service to distribute IP addresses to VMs", set the subnet to `192.168.100.0` with mask `255.255.255.0`, and applied the change.

I turned DHCP off on purpose. If VMware hands out addresses, they can change, and then the addresses in this README and in later firewall rules would be wrong. Fixed addresses make every later step easier to check.

<img src="docs/screenshots/01_vmnet_lab_net_config.png" alt="VMware Virtual Network Editor showing VMnet2 with DHCP disabled" width="600">

The Windows host gets `192.168.100.1` on this network through its VMware adapter. I check that from PowerShell in step 8.


#### 3. Creating target-node
I created `target-node` the same way as the other VMs: Ubuntu Server 24.04 LTS, user `ubuntu`, 2 vCPU, 4 GB RAM, 30 GB disk. I ticked **Install OpenSSH server** during installation, so I could SSH in from PowerShell instead of working in the VMware console.


#### 4. Two network adapters on each VM
`elk-siem-node`, `target-node` and `kali` each have two adapters:

- **Adapter 1:** Custom, VMnet2 (lab-net). Lab traffic only.
- **Adapter 2:** NAT. Package updates only.

I added them under **VM Settings > Add > Network Adapter** with the VM powered off. Lab-net is the first adapter, so its interface name is predictable. NAT is second, so the default route stays on it. I check that in step 8.

Why keep NAT at all? Without it the VMs cannot reach the Ubuntu and Elastic package repositories. Lab-net has no gateway configured, so the default route stays on NAT. I also do not want attack traffic to have a way out of the lab, which is why lab-net is the only route for `192.168.100.0/24`, and why Kali gets `never-default` on lab-net (step 7).


<img src="docs/screenshots/02_elk_siem_node_adapters.png" alt="Hardware settings on elk-siem-node with lab-net and NAT adapters" width="600">

#### 5. Stopping cloud-init from rewriting the network
Ubuntu Server uses cloud-init, and cloud-init rebuilds the network config at boot. That would wipe my static addresses on the next reboot. On both Ubuntu VMs I created a file that tells cloud-init to leave networking alone:


```bash
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

The file contains one line:

```yaml
network: {config: disabled}
```

Then I removed the netplan file cloud-init had generated, after my own file was in place:

```bash
sudo rm /etc/netplan/50-cloud-init.yaml
ls /etc/netplan/
```

Only `99-lab.yaml` are now remains.

#### 6. Static addresses with netplan
First I checked which interface name belongs to which adapter:

```bash
ip -br a
```

On my VMs, `ens33` is the NAT adapter and `ens37` is lab-net adapter. The giveaway was that the NAT adapter had already received an IP address via DHCP, while the other adapter was empty (no IP address assigned). Names can differ between hypervisor versions, so check your own output before copying anything from me.

I edited the network config over SSH. That worked because my SSH session ran over the NAT address, which did not change. If I had connected over lab-net, changing the lab-net address would have dropped the session in the middle of the change.

On `elk-siem-node` the file is `/etc/netplan/99-lab.yaml`:


```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: true
    ens37:
      dhcp4: false
      addresses:
        - 192.168.100.10/24
```

On `target-node` the file is the same, with `192.168.100.20/24` on ens37.

Netplan reads indentation strictly, so I used two spaces per level and no tabs. I restricted the file permissions and tested before applying:


```bash
sudo chmod 600 /etc/netplan/99-lab.yaml
sudo netplan try
```

`netplan try` applies the new config for a short time and waits for confirmation. If access is lost, it rolls back by itself. I used `try` and not `apply` the first time for that reason.


#### 7. Kali on lab-net
Kali uses NetworkManager, so I set the static address with `nmcli`. First I found the interface:

```bash
nmcli device status
```

Kali has `eth0` for NAT and `eth1` for lab-net. Then I created a connection named `lab-net`:

```bash
sudo nmcli con add type ethernet ifname eth1 con-name lab-net \
  ipv4.method manual ipv4.addresses 192.168.100.30/24 \
  ipv4.never-default yes
sudo nmcli con up lab-net
```

`ipv4.never-default yes` is the setting that matters here. Without it, NetworkManager can pick lab-net as the default route, and then internet access stops or traffic leaves by the wrong interface. With it, lab-net only handles `192.168.100.0/24`.

<img src="docs/screenshots/03_kali_ip_br_a.png" alt="ip -br a on Kali showing eth0 on NAT and eth1 on lab-net" width="600">

#### 8. Verification
Interfaces from `ip -br a`:

| VM | Interface | Role | State | Address |
|---|---|---|---|---|
| target-node | ens33 | NAT | UP | 192.168.95.155/24 |
| target-node | ens37 | lab-net | UP | 192.168.100.20/24 |
| elk-siem-node | ens33 | NAT | UP | 192.168.95.156/24 |
| elk-siem-node | ens37 | lab-net | UP | 192.168.100.10/24 |
| kali | eth0 | NAT | UP | 192.168.95.151/24 |
| kali | eth1 | lab-net | UP | 192.168.100.30/24 |

The routing table on `target-node` after the change:

```
default via 192.168.95.2 dev ens33 proto dhcp src 192.168.95.155 metric 100
192.168.95.0/24 dev ens33 proto kernel scope link src 192.168.95.155 metric 100
192.168.100.0/24 dev ens37 proto kernel scope link src 192.168.100.20
```

The default route goes out through NAT, and the lab network goes out through lab-net. `elk-siem-node` shows the same pattern with its own addresses.

<img src="docs/screenshots/04_ip_route_elk_siem_node.png" alt="ip route and interface addresses on elk-siem-node" width="600">

From each Ubuntu VM I pinged the other two lab addresses, and every ping got a reply with 0% packet loss. `curl` to the Ubuntu archive returned `HTTP/1.1 200 OK`, so NAT works for updates.

From the Windows host in PowerShell, all three lab addresses answered. SSH logged in to `target-node` and `elk-siem-node` without switching windows.

<img src="docs/screenshots/05_connectivity_from_windows.png" alt="Ping and SSH from the Windows host to lab-net" width="600">

Reboot test: after a reboot the addresses stayed the same: `ens37 UP 192.168.100.10/24`
Reboot test: after a reboot, `elk-siem-node` still had `ens37` UP on `192.168.100.10/24`. and `target-node` `192.168.100.20/24`.


#### 9. Result and assessment
- **Result:** All three VMs are on lab-net with fixed addresses. They reach each other and the Windows host, and they still reach the internet through NAT.
- **Assessment:** The network works, but it is flat, which is what I wanted for now. Any VM can reach any other VM on lab-net. That is fine for phases 2 to 10, and it is exactly what chapter 11 will change.
- **Problems I hit:** None that blocked the phase.

Phase 2 is base hardening on the two Ubuntu VMs, before Elasticsearch is installed.


----------------Work in Progress------------------
