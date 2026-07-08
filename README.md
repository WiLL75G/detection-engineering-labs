# Detection Engineering Labs

A collection of hands on detection engineering and SOC analysis labs, each documented as a Tier 1 style incident report with real attack execution, detection, and investigation. Every lab is built in a self contained home lab and mapped to MITRE ATT&CK.

## Labs

| Lab | Focus | Platform | MITRE |
| --- | --- | --- | --- |
| [01 Wazuh SSH Brute Force](01-wazuh-ssh-bruteforce/) | SSH brute force detection, failures to compromise | Wazuh 4.14.6 | T1110 |
| 02 ServiceNow ITSM | Incident ticketing and workflow (planned) | ServiceNow | pending |
| 03 Suricata IDS | Network intrusion detection (planned) | Suricata | pending |
| 04 PowerShell Investigation | Suspicious PowerShell analysis (planned) | Windows, Sysmon | pending |

## Home Lab

The labs run on a self contained lab: a macOS host, an Ubuntu Server victim, a Windows 11 endpoint, and a Kali Linux attacker, with detection tooling deployed across the environment. Each lab folder contains its own README and a screenshots directory with the supporting evidence.

## About

Built and documented by William, learning in public across detection engineering and SOC analysis. Each report favors real evidence over claims, and verifies platform specifics rather than assuming them.
