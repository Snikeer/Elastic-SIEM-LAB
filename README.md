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



# Elastic SIEM Lab

A hands-on lab for learning how a SIEM works in practice. You will build a small Elastic Stack, connect a monitored Linux host to it, attack that host from another virtual machine, and then write the detection rules that should catch those attacks. Everything runs in VMware on one Windows machine.

This is written as a guide, not only as documentation. Each step explains what we do and why, shows the commands, describes what the output should look like, and lists the mistakes people usually make. If you follow it from start to finish, you should get the same results, and you should understand why the parts fit together the way they do.

## Who this is for

You do not need to be a Linux expert. You should be comfortable opening a terminal, editing a text file, and reading an error message without panicking. Some networking knowledge helps in the segmentation chapter, but the chapter explains the basics as it goes.

If you have never used a SIEM before, read the glossary first. The rest of the guide uses those words without stopping to explain them again.

## What you need

- A Windows machine with VMware Workstation (Player is not enough for custom networks, use Pro or a version that supports host-only networks).
- Enough memory. The numbers in this guide assume a host with at least 32 GB of RAM. I run it on a host with 100 GB, but the lab fits on less if you lower the VM sizes. The minimum I would accept is 24 GB, and the Elastic VM should not go below 8 GB.
- Ubuntu Server 24.04 LTS ISO.
- Kali Linux VM (the official VMware image from kali.org works well).
- A GitHub account, if you want to follow the documentation part.

## How to read each step

Every step follows the same layout:

- **What we do:** one sentence on the action.
- **Why:** the reason for the step, and how it connects to the SIEM or the segmentation track.
- **Commands:** what to run, with a short description of each command.
- **Expected result:** what the output should look like.
- **If it goes wrong:** the most common problems and how to fix them.

Screenshots only appear where they show something you can check. If a step has no screenshot, the terminal output is the proof.

## Glossary

- **SIEM (Security Information and Event Management):** a system that collects logs from many machines, stores them in one place, and runs rules that flag suspicious activity.
- **Elasticsearch:** the search and storage engine that holds the logs.
- **Kibana:** the web interface where you search logs, view dashboards, and manage detection rules.
- **Fleet Server:** the component that manages Elastic Agents from one central place.
- **Elastic Agent:** a small program installed on each monitored host. It collects logs and sends them to Elasticsearch through Fleet.
- **Integration:** a ready-made package that tells the agent which logs to collect and how to read them.
- **Data stream:** how Elasticsearch stores time-based logs, so they can be searched and retained by policy.
- **Detection rule:** a condition that runs over incoming events and creates an alert when it matches.
- **ECS (Elastic Common Schema):** the standard field names Elastic uses, for example `source.ip` or `user.name`. Rules are easier to write when every log source uses the same names.
- **MITRE ATT&CK:** a public catalogue of attacker techniques. Each detection in this lab is linked to one or more techniques.
