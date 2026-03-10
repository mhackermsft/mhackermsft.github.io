---
title: 'Nation-State Cybercrime as a Service: Why Iranian-Linked Ransomware Campaigns Are a Real Threat to Government Agencies'
date: 2026-03-10T19:08:39+00:00
author: Mike Hacker
tags:
- Security
- Threat Intelligence
- Government
categories:
- Security
- Government
summary: Iranian state-sponsored threat actors are not just conducting espionage - they are actively brokering network access to ransomware affiliates, making US state and local government agencies prime targets in a dangerous new hybrid threat model.
draft: true
image: cover.png
---

## When Espionage Meets Extortion

For years, the conventional wisdom about nation-state cyber actors was straightforward: foreign governments send hackers to steal secrets. But evidence from Microsoft Threat Intelligence reveals that some nation-state actors, particularly those linked to Iran, have pivoted to a far more dangerous model - one where government-backed hackers sell or share network access with ransomware criminal groups, blurring the line between geopolitical operations and organized cybercrime.

This hybrid model, often called **Cybercrime as a Service (CaaS)**, poses a unique and elevated threat to US state and local government agencies. Understanding how it works, who is behind it, and how to defend against it is no longer optional for government IT leaders - it is operationally critical.

## The Threat Actors You Need to Know

Microsoft uses a weather-based naming taxonomy to track threat actors. Several Iranian-linked groups are actively operating in this space, and understanding their distinct roles helps security teams prioritize defenses.

### Lemon Sandstorm - The Access Broker

Perhaps the most directly relevant actor for state and local government is **Lemon Sandstorm** (also known as PIONEER KITTEN, Fox Kitten, RUBIDIUM, and UNC757). A joint FBI-CISA-DC3 advisory (AA24-241A, published August 2024) explicitly names this group as a significant threat to **schools, municipal governments, financial institutions, and healthcare facilities** in the United States.

What makes Lemon Sandstorm particularly dangerous is its business model: the FBI assesses that a significant portion of its US activity is aimed at gaining and maintaining network access - which it then sells on cybercrime marketplaces or hands off to ransomware affiliates. The group has been linked to collaborations with **NoEscape**, **RansomHouse**, and **ALPHV (BlackCat)** ransomware operations. Their role goes beyond providing a doorway: they work alongside affiliates to help lock victim networks and strategize on extortion approaches.

### Peach Sandstorm - The Infrastructure Abuser

**Peach Sandstorm** (formerly HOLMIUM/APT33) operates on behalf of Iran's Islamic Revolutionary Guard Corps (IRGC) and has historically focused on intelligence gathering. Its [2024 campaign deploying custom "Tickler" malware](https://www.microsoft.com/en-us/security/blog/2024/08/28/peach-sandstorm-deploys-new-custom-tickler-malware-in-long-running-intelligence-gathering-operations/) targeted **federal and state government sectors** in the United States directly.

The group conducts **password spray attacks** against government and defense targets. Once inside, they have been observed creating fraudulent Azure subscriptions using compromised credentials to host command-and-control infrastructure, hiding malicious activity within legitimate cloud services.

### Mango Sandstorm and Storm-1084 - Destruction Behind a Ransomware Mask

Linked to Iran's Ministry of Intelligence and Security (MOIS), **Mango Sandstorm** (formerly MERCURY) has conducted what [Microsoft describes as "destructive operations" disguised as ransomware campaigns](https://www.microsoft.com/en-us/security/blog/2023/04/07/mercury-and-dev-1084-destructive-attack-on-hybrid-environment/). In documented attacks, these actors gained initial access, then handed off to partner cluster Storm-1084 to perform mass deletion of cloud resources - including virtual machines, storage accounts, and virtual networks - while sending threatening emails to victims.

This distinction is critical: what looks like ransomware demanding payment may actually be a cover for irreversible data destruction. For government agencies managing critical records, permitting systems, or public safety data, this scenario represents an existential operational threat.

### Mint Sandstorm - Ransomware as a Revenue Stream

**Mint Sandstorm** (formerly PHOSPHORUS/CHARMING KITTEN/APT35) includes a sub-group [Microsoft tracked as Storm-0270](https://www.microsoft.com/en-us/security/blog/2022/09/07/profiling-dev-0270-phosphorus-ransomware-operations/) that conducts ransomware campaigns alongside state-directed operations. This group exploits unpatched vulnerabilities in **Microsoft Exchange** and **Fortinet VPN appliances**, then uses built-in Windows tools and **BitLocker** to encrypt victim data - with observed attacks moving from initial compromise to ransom note in as little as 48 hours.

## Why This Matters for Government

US state and local government agencies occupy a uniquely vulnerable position in this threat landscape:

**High-value data, constrained resources.** Government agencies hold sensitive citizen data, legal records, and financial systems - yet many operate with security budgets that lag far behind the private sector. This asymmetry is precisely what access brokers like Lemon Sandstorm exploit.

**Legacy systems and delayed patching cycles.** Nation-state actors specifically target long-known, unpatched vulnerabilities. Government procurement and change management processes can create windows of exposure that persist for months.

**The ransomware-espionage overlap.** Even when a ransomware attack appears financially motivated, it may have originated from a nation-state actor. Incident response cannot stop at restoring systems - agencies must also assume data has been exfiltrated for intelligence purposes.

**Local government is explicitly named.** The August 2024 CISA advisory (AA24-241A) specifically calls out **municipal governments** among confirmed target sectors for Iranian-linked actors. This is not a hypothetical threat.

**The CaaS model lowers the barrier to attack.** When a nation-state actor sells access to your network, any number of ransomware groups can become your adversary - sophistication required to attack you goes down while your exposure surface goes up.

## How Microsoft Helps Government Defend Against These Threats

### Stop Password Spray Attacks at the Door

Peach Sandstorm's primary initial access method - password spraying - is effectively mitigated by **Microsoft Entra ID's** [Conditional Access policies and Multi-Factor Authentication](https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview). For GCC tenants using M365, enabling MFA through Microsoft Entra ID is a foundational control that directly counters this threat vector.

### Detect Lateral Movement and Unusual Behavior

**[Microsoft Defender for Endpoint](https://learn.microsoft.com/en-us/defender-endpoint/microsoft-defender-endpoint)** monitors for the living-off-the-land techniques that Iranian actors favor - abuse of native Windows tools like `wmic`, `whoami`, `netstat`, and `nltest`. These behavioral signals are correlated across your environment in **Microsoft Defender XDR**, which integrates endpoint, identity, email, and cloud app signals into a unified view.

### Hunt Threats with Microsoft Sentinel

**[Microsoft Sentinel](https://learn.microsoft.com/en-us/azure/sentinel/detect-threats-built-in)** provides SIEM and SOAR capabilities that government agencies can use to deploy analytics rules tuned to Iranian actor TTPs. The Sentinel Content Hub includes threat intelligence-driven rule templates based on known attack patterns. Security teams can also use **[custom analytics rules](https://learn.microsoft.com/en-us/azure/sentinel/detect-threats-custom)** written in KQL to hunt for specific indicators of compromise tied to these groups.

### Protect Cloud Resources and Manage Patch Exposure

**Microsoft Defender for Cloud** monitors Azure resources for misconfigurations and anomalous activity - directly relevant given Peach Sandstorm's use of Azure infrastructure for command-and-control staging. Government agencies should ensure Defender for Cloud is enabled across both Azure Commercial and Azure Government environments. **Microsoft Defender Vulnerability Management** complements this by providing continuous visibility into unpatched, internet-facing assets, enabling teams to prioritize patches based on active exploitation data rather than CVSS scores alone.

## What Government IT Leaders Should Do Now

1. **Enforce MFA on all accounts** - particularly for email, VPN, and administrative access. This single control disrupts the password spray attacks that Peach Sandstorm relies on.
2. **Audit internet-facing services** - identify and immediately patch Exchange servers, Fortinet VPN appliances, and any perimeter devices with known unpatched vulnerabilities.
3. **Deploy Microsoft Sentinel with Iranian actor threat intelligence rules** - the Sentinel Content Hub includes rule templates aligned to documented Iranian actor TTPs.
4. **Assume breach for ransomware incidents** - treat any ransomware event as a potential nation-state intrusion. Engage Microsoft's incident response resources and notify the FBI's local field office as recommended by CISA advisory AA24-241A.
5. **Review Azure subscription activity** - look for unexpected new subscriptions or resource creation, as Peach Sandstorm has been documented creating fraudulent Azure infrastructure from compromised credentials.

## The Bottom Line

The threat from Iranian-linked cyber actors is no longer limited to federal agencies or defense contractors. Confirmed targeting of municipal governments by groups that actively broker network access to ransomware affiliates means every city and county IT department now sits in a threat landscape once reserved for national security targets.

The most effective defenses - MFA, patching, behavioral threat detection, and SIEM coverage - are not exotic or prohibitively expensive. Microsoft's integrated security platform, available through existing M365 and Azure investments, provides the tooling needed to detect and disrupt these attacks at multiple stages of the kill chain. The risk is real, the tools exist, and the time to act is now.

---

*To learn more about Microsoft's threat actor naming taxonomy and Iranian-linked groups, see the [Microsoft Threat Actor Naming](https://learn.microsoft.com/en-us/defender-xdr/microsoft-threat-actor-naming) page on Microsoft Learn. For MFA and Conditional Access deployment guidance, visit the [Microsoft Entra ID Conditional Access documentation](https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview).*
