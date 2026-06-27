# Day 09 – Deploying Suricata IDS on pfSense

## What I did today

Added Suricata to the pfSense box to get actual network-level intrusion detection going in the lab. Up until now I had Wazuh watching the hosts (HIDS side), but nothing was watching the wire itself. Suricata fills that gap, it sits on the LAN interface and inspects every packet flowing between my VMs.

## IDS vs IPS, quickly

- **IDS** = watches traffic, raises an alert when something matches a bad pattern. Doesn't block anything.
- **IPS** = same detection, but can actually drop/block the traffic inline.

Suricata can do either; I'm running it in IDS mode for now since I just want visibility before I start blocking things in my own lab.

## Signature-based vs anomaly-based detection

**Signature-based** : matches traffic against a known list of attack patterns (think: an Nmap scan signature, a known malware C2 beacon, a SQLi payload). Fast and accurate, but only for things someone has already written a rule for.

**Anomaly-based** : builds a baseline of "normal" traffic and flags deviations from it. Can catch novel attacks, but throws more false positives since "normal" is fuzzy.

## NIDS vs HIDS

- **NIDS** (Network Intrusion Detection system) : watches packets on the wire. Suricata, Snort, Zeek. Usually deployed on a firewall, a network TAP.
- **HIDS** (Host Intrusion Detection system) : watches what's happening _on_ a machine: logs, file integrity, processes, the registry on Windows. This is what Wazuh agents do.

Good to finally have both sides covered in the lab.

## Current lab topology

```
Internet
   │
   ▼
pfSense
   │
LAN (10.0.1.0/24)
   │
 ├── Kali (attacker)
 ├── Ubuntu
 ├── Windows target
 ├── Jump host
 └── Wazuh
```

Everything talks through the pfSense LAN interface, which makes it the natural place to plug Suricata in.

## Installing and configuring Suricata

Installed the Suricata package on pfSense and pointed it at the **LAN interface (vtnet1)** that's the one all my VM-to-VM and VM-to-internet traffic crosses, so it's the most useful place to watch.

![alt text](<./Screenshots/Screenshot 2026-06-27 122737.png>)
Pulled in:

- ET Open rules
- Feodo Tracker rules
- Abuse.ch SSL blacklist

Interface settings:

- Promiscuous mode on
- EVE JSON logging on
- HTTP / DNS / TLS / SSH / file logging all on
- Detection profile: Medium
- Run mode: AutoFP

Also disabled hardware checksum offloading, TCP segmentation offloading, and large receive offloading , both Suricata's own docs and Netgate recommend this on virtualized NICs, otherwise the packets Suricata sees on the wire don't match what's actually being sent, which breaks inspection.

## Testing it

Generated some traffic from Kali to see if Suricata would pick it up:

```bash
ping 10.0.1.20
nmap -sS -T5 -A 10.0.1.20
curl http://testmyids.com
```

![alt text](<./Screenshots/Screenshot 2026-06-27 124449.png>)

Checked `eve.json` afterward and traffic was definitely being seen and parsed DNS queries/responses, HTTP requests, TLS sessions, fileinfo events all showing up correctly. So packet capture and protocol decoding are both working fine.

## The problem: no signature alerts

![alt text](<./Screenshots/Screenshot 2026-06-27 123618.png>)

Here's where I got stuck. Protocol events are logging perfectly in `eve.json`, but I'm not getting actual **signature-based alerts** not even for the Nmap scan, which should be an easy, well-known signature hit.

I went through:

- Confirmed the Suricata service is actually running
- Confirmed rule updates pulled successfully
- Re-checked LAN interface binding
- Re-checked logging config
- Enabled `emerging-scan.rules` specifically (this should cover the Nmap SYN scan)

Still nothing in the alert log, even though the HTTP/DNS/TLS events are clearly there. So traffic visibility works, but detection isn't firing.

## My note (debugging in progress)

> Installed Suricata, eve.json is logging fine HTTP, DNS, TLS, fileinfo are all showing up. But I'm not getting any actual alerts out of it, even running an Nmap scan against the target which should trip an easy signature. Rules are updated, scan rule category is enabled, interface and logging look right on paper. Still digging into why signatures aren't matching even though Suricata is clearly seeing the traffic.

Probably one of:

- Rule categories enabled in the GUI but not actually written into the running `suricata.yaml`
- Wrong interface/promiscuous mode interaction with how pfSense passes traffic to the Suricata process
- AutoFP run mode behaving oddly with how I've got vtnet1 set up
- A signature `action` set to "alert" not actually being applied because the policy/profile is filtering it out before it logs

Next step is to manually grep the loaded rule file for an Nmap/scan signature and confirm it's actually active, rather than trusting the GUI checkbox.

## Key takeaway

Big lesson today: **visibility ≠ detection**. Seeing protocol events in `eve.json` only proves Suricata is capturing and decoding traffic correctly. Whether it raises an _alert_ depends entirely on whether a loaded rule actually matches that traffic. I had assumed once I saw events flowing, alerts would just follow they don't.

## Next steps

- [ ] Figure out why signature alerts aren't firing
- [ ] Confirm scan detection rules are actually loaded and active
- [ ] Get a clean Nmap-triggered alert end to end
- [ ] Pipe Suricata's `eve.json` into Wazuh
- [ ] Get NIDS alerts showing in the Wazuh dashboard
- [ ] Write up the full NIDS → SIEM integration once it's working

## Status

- [x] Suricata installed
- [x] Rules downloaded
- [x] LAN interface monitoring configured
- [x] Traffic inspection confirmed
- [x] EVE JSON logging working
- ⚠️ Signature alerts not generating still troubleshooting

