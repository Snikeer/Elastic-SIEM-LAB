## Lab Design

The lab runs entirely inside VMware Workstation. The Windows host runs only the browser and the SSH client. The work is split into two tracks. The main track is the Elastic SIEM, built the way a SOC would use it. The second track is network segmentation, which is added after the SIEM works.

### Track 1: SIEM on a flat lab network

| VM | IP (lab-net) | Role |
|---|---|---|
| Windows host | 192.168.100.1 | Browser for Kibana, SSH client |
| `elk-siem-node` | 192.168.100.10 | Elasticsearch, Kibana, Fleet Server |
| `target-node` | 192.168.100.20 | Monitored host, DVWA in Docker, Elastic Agent |
| `kali` | 192.168.100.30 | Attacker |

The flat network comes first so that problems with the SIEM are not mixed up with problems with firewall rules. Each VM also has a NAT adapter, used only for package updates.

### Track 2: Segmentation

A fourth VM, `fw-node`, is added as a router and firewall with nftables. It splits the lab into four zones and enforces default deny between them. Its logs are shipped to the SIEM, so segmentation becomes a data source as well.

| Zone | Network | Hosts |
|---|---|---|
| mgmt | 10.10.10.0/24 | Windows host, fw-node |
| attack | 10.10.20.0/24 | kali |
| dmz | 10.10.30.0/24 | target-node |
| security | 10.10.40.0/24 | elk-siem-node |

Allowed flows: the Windows host reaches all zones over SSH, and the security zone over Kibana. The attack zone reaches the dmz zone. The dmz zone reaches the security zone on Fleet and Elasticsearch ports only. The attack zone has no path to the security zone.

### Phases
0. Design (this section)
1. Virtual machines and flat network
2. Base hardening
3. Elasticsearch and Kibana
4. Fleet Server and Elastic Agent
5. Baseline data collection
6. DVWA target
7. Attack simulation from Kali
8. Detection rules, tested against the attacks
9. Dashboard and alerting
10. Summary and lessons learned
11. Segmentation: fw-node, zones, default deny, firewall logs in the SIEM
