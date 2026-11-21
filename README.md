# Elastic SIEM Lab

A hands-on lab for learning how a SIEM works in practice. I will build a small Elastic Stack, connect a monitored Linux host to it, attack that host from another virtual machine, and write the detection rules that should catch those attacks. Everything runs in VMware on one Windows machine.

This is written as a guide, not only as documentation. Each step explains what we do and why, shows the commands, describes what the output should look like, and lists the mistakes people usually make. If you follow it from start to finish, you should get the same results, and you should understand why the parts fit together the way they do.

## Who this is for

You do not need to be a Linux expert. You should be comfortable opening a terminal, editing a text file, and reading an error message without giving up. Some networking knowledge helps in the segmentation chapter, and that chapter explains the basics as it goes.

If you have never used a SIEM before, read the glossary first. The rest of the guide uses those words without explaining them again.

## What you need

A Windows machine with VMware Workstation Pro. VMware Player does not support the custom host-only networks this lab uses.

Memory is the main constraint. This guide is written for a host with 32 GB of RAM, which fits the VM sizes in the design table below with some room left for Windows. The lab can run on 24 GB if you give the Elastic VM 8 GB and keep the other VMs small. I have not tested the smaller setup yet, so treat it as a plan, not a guarantee. The Elastic VM should not go below 8 GB.

You also need the Ubuntu Server 24.04 LTS ISO, the official Kali Linux VMware image from kali.org, and a GitHub account if you want to follow the documentation part.

## How to read each step

Every step follows the same layout. The first line says what we do. The next paragraph explains why, and how the step connects to the SIEM or to the segmentation track. After that come the commands, with a short description of what each one does. Then comes the expected result, so you know the step worked before moving on. The last part covers the most common things that go wrong and how to fix them.

Screenshots only appear where they show something you can check. If a step has no screenshot, the terminal output is the proof.

## Glossary

**SIEM (Security Information and Event Management):** a system that collects logs from many machines, stores them in one place, and runs rules that flag suspicious activity.

**Elasticsearch:** the search and storage engine that holds the logs.

**Kibana:** the web interface where you search logs, view dashboards, and manage detection rules.

**Fleet Server:** the component that manages Elastic Agents from one central place.

**Elastic Agent:** a small program installed on each monitored host. It collects logs and sends them to Elasticsearch through Fleet.

**Integration:** a ready-made package that tells the agent which logs to collect and how to read them.

**Data stream:** how Elasticsearch stores time-based logs, so they can be searched and kept according to a retention policy.

**Detection rule:** a condition that runs over incoming events and creates an alert when it matches.

**ECS (Elastic Common Schema):** the standard field names Elastic uses, for example `source.ip` or `user.name`. Rules are easier to write when every log source uses the same names.

**MITRE ATT&CK:** a public catalogue of attacker techniques. Each detection in this lab is linked to one or more of them.

## Lab design

### Why two tracks

The main track is the SIEM. The first part of the lab uses one flat network, so when something breaks you know the problem is in the SIEM and not in a firewall rule. Network segmentation comes later, as chapter 11. By then logs are already flowing, so each firewall rule can be tested against real data.

### Track 1: SIEM on a flat lab network

| VM | IP on lab-net | RAM | vCPU | Disk | Role |
|---|---|---|---|---|---|
| Windows host | 192.168.100.1 | (host) | (host) | (host) | Browser for Kibana, SSH client |
| `elk-siem-node` | 192.168.100.10 | 16 GB | 4 | 80 GB | Elasticsearch, Kibana, Fleet Server |
| `target-node` | 192.168.100.20 | 4 GB | 2 | 30 GB | Monitored host, DVWA in Docker, Elastic Agent |
| `kali` | 192.168.100.30 | 8 GB | 2 | existing | Attacker |

All lab VMs sit on one host-only network called `lab-net`, `192.168.100.0/24`. The Windows host gets `.1` through its VMware adapter, so you can open Kibana in the browser and SSH into every VM from PowerShell or VS Code without switching windows.

Each VM also has a second network adapter on NAT. It is used only for package updates, so the machines can reach the Ubuntu and Elastic repositories. Attack traffic never uses that adapter, which is why it does not affect the lab.

### Track 2: Segmentation (chapter 11)

Real companies do not run everything on one network. A SOC usually sees traffic from several zones, and firewall logs are one of the most useful data sources it has. In this track, a fourth VM, `fw-node`, becomes a router and firewall running nftables. It splits the lab into four zones and denies traffic between them by default. Its logs are sent to the same SIEM, so the firewall becomes a data source too.

| Zone | Network | Hosts |
|---|---|---|
| mgmt | 10.10.10.0/24 | Windows host, fw-node |
| attack | 10.10.20.0/24 | kali |
| dmz | 10.10.30.0/24 | target-node |
| security | 10.10.40.0/24 | elk-siem-node |

The Windows host can reach every zone over SSH, and it can reach Kibana in the security zone. The attack zone can reach the dmz zone, which is the point of the exercise. The dmz zone can reach Fleet and Elasticsearch in the security zone, on those ports only, because agents always start the connection. The attack zone has no path to the security zone at all, so Kali cannot see the SIEM.

## Phases

- [ ] Phase 0: Design (this section)
- [ ] Phase 1: Virtual machines and flat network
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

## Step-by-step guide

The steps are added as each phase is completed. Phase 1 comes first.
