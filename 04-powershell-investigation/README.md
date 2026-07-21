# Suspicious PowerShell Investigation on Windows (Sysmon and Script Block Logging, MITRE T1059.001)

An opaque base64 blob on the command line, recovered in full plaintext one second later from a child-process log event. This is the lab that separates "PowerShell ran" from "here is exactly what it did", and the difference is entirely down to logging that was turned on before the attack, not after.

## At a Glance

| Field | Detail |
| --- | --- |
| Work Type | Endpoint forensics, Windows |
| Endpoint | JAMES-VM, Windows 11, PowerShell 5.1 |
| User context | james-vm\william, local Administrator |
| Telemetry | Sysmon EID 1, PowerShell Script Block Logging EID 4104 |
| Behaviors | Encoded command, download cradle, discovery sequence |
| MITRE | T1059.001, T1105, T1033, T1082, T1087 |
| Centerpiece | Base64 command recovered in plaintext from 4104 |
| Date | 10 July 2026 |

## What This Is

A Windows endpoint forensics investigation. Suspicious PowerShell simulating real attacker behavior is executed on JAMES-VM and then reconstructed from Sysmon process telemetry and PowerShell Script Block Logging.

The investigation recovers the deobfuscated content of a base64-encoded command, identifies a download cradle, and reconstructs a post-compromise discovery sequence, all from the endpoint logs. It closes the lab set with the skill the other three do not cover: not detecting that something happened, but reconstructing what.

## Incident Summary

On 09 July 2026, three distinct attacker behaviors were run on host JAMES-VM under the local administrator account william: a base64-encoded PowerShell command, a download cradle using the classic IEX DownloadString pattern, and a discovery sequence enumerating the user, privileges, hostname, and local accounts.

Windows PowerShell Script Block Logging (Event ID 4104) captured the deobfuscated script content of all three, including the plaintext hidden inside the encoded command. Sysmon (Event ID 1) independently recorded the process creation. The activity maps to MITRE T1059.001 and related discovery techniques.

The finding that carries the lab is the recovery of the encoded command in plaintext: obfuscation on the command line does not survive properly configured logging.

## Affected System

| Attribute | Value |
| --- | --- |
| Endpoint | JAMES-VM (Windows 11) |
| User context | james-vm\william (local Administrator) |
| Telemetry (host) | Sysmon Operational log, Event ID 1 |
| Telemetry (PowerShell) | PowerShell Operational log, Event ID 4104 |
| PowerShell version | 5.1 |
| Date | 10 July 2026 |

## Investigation Methodology

### 1. Baseline and telemetry verification

Before generating any activity, both telemetry sources were confirmed. You cannot investigate what was not logged, so this is the most important step in an endpoint forensics exercise.

![Sysmon logging confirmed](screenshots/01_sysmon_logging_confirmed.png)

![Script Block Logging enabled](screenshots/02_scriptblock_logging_enabled.png)

**SOC Observations**

Sysmon was confirmed active and producing Event ID 1 (Process Create) records with recent timestamps. PowerShell Script Block Logging was confirmed enabled in the registry (EnableScriptBlockLogging set to 1), meaning PowerShell would record the content of every script block it executed, including the deobfuscated form of any encoded command. This step is not preamble, it is the entire case being made possible before the attack runs.

### 2. Attack simulation

Three attacker behaviors were executed, each mapped to MITRE ATT&CK and each harmless in effect but faithful to real technique.

![Attacks executed](screenshots/03_attacks_executed.png)

**SOC Observations**

Behavior one was a base64-encoded command run via powershell.exe -EncodedCommand. On the command line this appears only as an opaque base64 blob, exactly how attackers hide intent. Behavior two was a download cradle, IEX (New-Object Net.WebClient).DownloadString(...), pointed at localhost so it failed harmlessly while still generating the recognizable cradle signature. Behavior three was a discovery sequence: whoami, group enumeration, hostname, and local account listing.

The discovery step matters beyond the technique tag. It confirmed the running account is a member of the local Administrators group and enumerated other enabled accounts on the host, which in a real incident is the difference between a foothold and a foothold that already owns the box.

### 3. Investigation: recovering the scripts

The PowerShell Operational log was queried for Event ID 4104 to recover the actual script content of each behavior.

![Investigation of 4104 events](screenshots/04_investigation_4104_cradle_discovery.png)

**SOC Observations**

The 4104 events captured the full plaintext of the download cradle and the discovery sequence. Routine PowerShell internal script blocks (Set-StrictMode, prompt) also appear in the log and were filtered out as noise.

A real investigative lesson surfaced here: the encoded command had scrolled out of the initial 20-event window, so the search had to be widened to locate it. That is not a footnote, it is the reminder that log scoping decides what you find, and a too-narrow window hides the one event that matters.

### 4. Defeating obfuscation

A targeted search recovered the encoded command. This is the centerpiece finding.

![Deobfuscation proof](screenshots/05_deobfuscation_proof.png)

**SOC Observations**

Two linked 4104 events tell the story. The first captured the launcher the analyst would see, ending in powershell.exe -EncodedCommand followed by a base64 blob. One second later a separate 4104 event, generated by the child process, captured the decoded plaintext: Write-Output "Simulated encoded command executed - MITRE T1059.001".

The attacker obfuscated the command with base64, but Script Block Logging recorded the real command underneath. No manual base64 decoding was required, the logging did it. That is the whole argument for enabling 4104 across a fleet: it turns obfuscation from a wall into a speed bump.

## Indicators of Compromise

| Type | Indicator |
| --- | --- |
| Host | JAMES-VM |
| Account | james-vm\william (local Administrator) |
| Encoded command | powershell.exe -EncodedCommand (base64) |
| Download cradle | IEX (New-Object Net.WebClient).DownloadString |
| Discovery | whoami, whoami /groups, Get-LocalUser |

## MITRE ATT&CK

| Tactic | Technique | ID |
| --- | --- | --- |
| Execution | Command and Scripting Interpreter: PowerShell | T1059.001 |
| Command and Control | Ingress Tool Transfer | T1105 |
| Discovery | System Owner/User Discovery | T1033 |
| Discovery | System Information Discovery | T1082 |
| Discovery | Account Discovery | T1087 |

## Findings

Three attacker behaviors were executed and fully reconstructed from endpoint telemetry.

The most significant finding is that the base64-encoded command was recovered in plaintext through Script Block Logging, proving obfuscation on the command line does not defeat properly configured logging. The launcher and the decoded child-process block, one second apart, are the proof.

The discovery sequence revealed the account in use was a local administrator, which raises the severity of a real incident because it means the attacker already held high privileges rather than needing to escalate to them.

## Response

In a production incident the response would be to isolate the host, capture the full 4104 and Sysmon timeline for the session, identify any successful outbound connections from the download cradle, review the enumerated accounts for signs of lateral movement or privilege abuse, and reset the compromised account.

The presence of Script Block Logging and Sysmon is itself a control worth confirming across the fleet, since this investigation was only possible because both were enabled in advance. The response to this incident includes making sure the next one is just as recoverable.

## The SOC Angle

The lesson that stuck is that logging configuration is a prerequisite for investigation, not an afterthought.

The entire case rested on two settings being enabled before the activity happened: Sysmon process auditing and PowerShell Script Block Logging. With them on, an encoded command that looks like meaningless base64 on the command line was recovered in full plaintext, and a multi-step attack was reconstructed end to end from logs alone. With them off, the same activity leaves an analyst staring at a blob and a shrug.

Endpoint forensics is less about clever decoding and more about ensuring the right telemetry exists, then reading it carefully and separating attacker activity from routine system noise. The clever part was already done, by whoever turned 4104 on.

## What This Demonstrates

Verifying endpoint telemetry (Sysmon and Script Block Logging) before generating activity.

Simulating three real attacker techniques safely: encoded command, download cradle, and discovery.

Querying and interpreting Event ID 4104 and Sysmon Event ID 1.

Recovering the deobfuscated content of a base64-encoded command from the logs.

Reconstructing a post-compromise discovery sequence and identifying an administrator account in use.

Distinguishing attacker script blocks from routine PowerShell internal noise.

Widening log scope to recover an event that fell outside the initial window.

## Repository Structure

```
04-powershell-investigation/
├── README.md
└── screenshots/
    ├── 01_sysmon_logging_confirmed.png
    ├── 02_scriptblock_logging_enabled.png
    ├── 03_attacks_executed.png
    ├── 04_investigation_4104_cradle_discovery.png
    └── 05_deobfuscation_proof.png
```

## Conclusion

This project demonstrated Windows endpoint forensics for suspicious PowerShell. Attacker behaviors were simulated and then fully reconstructed from Sysmon and PowerShell Script Block Logging, including recovering the plaintext of an obfuscated command. It closes the lab set with a host-based investigation skill that complements the network and host detection built in the earlier projects: seeing not just that PowerShell ran, but exactly what it did.

---

[![LinkedIn](https://img.shields.io/badge/LinkedIn-WilliamInCyber-blue?style=flat&logo=linkedin)](https://linkedin.com/in/WilliamInCyber)
[![X](https://img.shields.io/badge/X-@WilliamInCyber-black?style=flat&logo=x)](https://x.com/WilliamInCyber)
