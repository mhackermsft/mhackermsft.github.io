---
title: 'Secure Boot Certificate Expiration: A Hands-On Readiness Guide for Government IT Teams Running Mixed Windows and Linux Fleets'
date: 2026-06-19T13:44:18+00:00
author: Mike Hacker
tags:
- Security
- How To
categories:
- Security
summary: The first 2011-era Microsoft Secure Boot certificate in the current expiration table expires on June 24, 2026. Here is how government IT teams inventory, pilot, and roll out the 2023 certificate updates across Windows, dual-boot, and Linux endpoints.
draft: false
image_prompt: A cinematic wide shot inside a secure, modern government data center at dusk, rows of dark server racks receding into soft blue shadow. A luminous golden chain of interlocking links of light threads horizontally through the racks, symbolizing an unbroken chain of trust. Near the center, two gloved hands carefully lift out a single tarnished, cracked old link and replace it with a freshly forged, brightly glowing new link, sparks of light around the seam. Cool blue ambient lighting contrasts with the warm gold of the chain. The mood is precise, secure, and forward looking, conveying renewal before a critical deadline. Shallow depth of field, volumetric light, high detail. No text, letters, numbers, logos, or writing anywhere in the image.
audio: audio.mp3
image: cover.png
---

Secure Boot has protected the earliest moments of the device boot sequence for more than a decade. Microsoft documents that Secure Boot certificates issued in 2011 begin expiring in June 2026. Microsoft is replacing them with 2023 Secure Boot certificates so devices can continue to receive boot-level protections after the older certificates age out.

For government IT teams managing UEFI endpoints, servers, and mixed Windows and Linux workloads, this is a planned operational milestone. It is not a panic event, but it does deserve a staged plan that covers firmware currency, BitLocker recovery readiness, inventory, pilots, and deployment rings.

## What Is Actually Expiring

Secure Boot is part of the UEFI firmware model. It establishes a chain of trust from firmware through the boot loader and into the operating system so that only trusted early boot code can run. That trust is enforced through firmware-resident variables:

- **PK (Platform Key)**: the root of trust, typically owned by the OEM.
- **KEK (Key Enrollment Key)**: the keys allowed to update the signature databases. Microsoft documentation also refers to KEK as Key Exchange Key.
- **DB (Allowed Signature Database)**: trusted certificates and signatures that are allowed to boot.
- **DBX (Revoked Signature Database)**: revoked signatures that are explicitly blocked.

Microsoft's current certificate table identifies these 2011 certificates and replacements:

| Expiring certificate | Expiration date | 2023 replacement | Store | Purpose |
| --- | --- | --- | --- | --- |
| Microsoft Corporation KEK CA 2011 | June 24, 2026 | Microsoft Corporation KEK 2K CA 2023 | KEK | Signs updates to DB and DBX |
| Microsoft Windows Production PCA 2011 | October 19, 2026 | Windows UEFI CA 2023 | DB | Signs the Windows boot loader |
| Microsoft UEFI CA 2011 | June 27, 2026 | Microsoft UEFI CA 2023 | DB | Signs third-party boot loaders and EFI applications |
| Microsoft UEFI CA 2011 | June 27, 2026 | Microsoft Option ROM UEFI CA 2023 | DB | Signs third-party option ROMs |

These updates are applied to Secure Boot variables through supported Windows servicing and management channels, with OEM firmware updates where required. They are not a general instruction to reflash every device as a one-off project.

Microsoft's current overview is **Windows Secure Boot certificate expiration and CA updates**: [https://support.microsoft.com/en-us/topic/windows-secure-boot-certificate-expiration-and-ca-updates-7ff40d33-95dc-4c3c-8725-a9b95457578e](https://support.microsoft.com/en-us/topic/windows-secure-boot-certificate-expiration-and-ca-updates-7ff40d33-95dc-4c3c-8725-a9b95457578e). Microsoft Learn also provides troubleshooting guidance at [https://learn.microsoft.com/en-us/troubleshoot/windows-client/windows-security/update-secure-boot-certificates](https://learn.microsoft.com/en-us/troubleshoot/windows-client/windows-security/update-secure-boot-certificates).

## What Happens If You Do Nothing

Expiration does not automatically brick devices. Microsoft states that devices without the 2023 certificates can continue to start and operate normally, and standard Windows updates can continue to install.

The risk is forward protection. Devices that do not receive the 2023 certificates may no longer be able to receive future Secure Boot protections for early boot components, including Windows Boot Manager, Secure Boot databases, revocation lists, and mitigations for newly discovered boot-level vulnerabilities. Over time, that erodes the boot-integrity posture that many agencies depend on for device trust, BitLocker hardening, and audit confidence.

Microsoft also documents higher-risk scenarios for devices with outdated firmware or certificate updates that do not apply correctly. Symptoms can include Secure Boot validation errors, BitLocker recovery prompts, startup hangs, and devices failing to boot. That is why firmware readiness and pilot rings matter.

## Step 1: Inventory Affected Devices

Run these checks from an elevated PowerShell session. Start by confirming whether Secure Boot is enabled:

```powershell
Confirm-SecureBootUEFI
```

The cmdlet returns `True` when Secure Boot is enabled, `False` when it is disabled, and an error on unsupported BIOS or non-UEFI platforms.

On devices updated with the April 14, 2026 monthly cumulative update or later, use the `-Decoded` parameter to inspect DB contents in a readable form:

```powershell
$dbSubjects = (Get-SecureBootUEFI -Name db -Decoded).Subject

[pscustomobject]@{
    Hostname                 = $env:COMPUTERNAME
    SecureBootEnabled        = (Confirm-SecureBootUEFI)
    WindowsCA2023InDB        = [bool]($dbSubjects -match 'Windows UEFI CA 2023')
    ThirdPartyBootCA2023InDB = [bool]($dbSubjects -match 'Microsoft UEFI CA 2023')
    OptionRomCA2023InDB      = [bool]($dbSubjects -match 'Microsoft Option ROM UEFI CA 2023')
}
```

Microsoft also exposes deployment status through a registry value. The `UEFICA2023Status` value is under the Secure Boot servicing key, and a successful deployment reports `Updated`:

```powershell
Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecureBoot\Servicing' `
    -Name 'UEFICA2023Status' -ErrorAction SilentlyContinue
```

Finally, review the System event log for Secure Boot update events. Event ID 1808 indicates that the required 2023 Secure Boot certificates have been applied to firmware and that the boot manager has been updated to the boot manager signed by `Windows UEFI CA 2023`. Event IDs 1801, 1795, and 1796 are useful error or incomplete-state signals:

```powershell
Get-WinEvent -FilterHashtable @{ LogName = 'System'; Id = 1808, 1801, 1795, 1796 } `
    -MaxEvents 50 |
    Select-Object TimeCreated, Id, LevelDisplayName, ProviderName, Message
```

Roll these signals into Microsoft Intune, Configuration Manager, endpoint management reporting, or a scheduled inventory script so leadership can see update status by model, firmware version, and device class.

## Step 2: Update Firmware First

The most common rollout risk is stale UEFI firmware. Before broad certificate deployment, update OEM firmware, especially for older models and long-lived government hardware. Microsoft guidance for Secure Boot updates calls out firmware compatibility as a gating factor for DB, DBX, and KEK changes.

For agencies with long refresh cycles, treat firmware currency as its own workstream. Devices several years behind on firmware are the ones most likely to create BitLocker recovery prompts or Secure Boot update failures during a certificate rollout.

## Step 3: Pilot Before You Deploy

Build a pilot ring that represents the fleet, not just the newest laptops in IT. Include:

- Multiple OEMs and device models.
- Different firmware versions, including older supported baselines.
- BitLocker-enabled devices.
- Dual-boot workstations and Linux systems, if they exist in the environment.
- Servers or appliances that rely on Secure Boot.

For each pilot device, confirm that the certificate status reaches `Updated`, the device boots cleanly, and users do not receive unexpected BitLocker recovery prompts. Before any deployment ring, verify that BitLocker recovery keys are escrowed in Microsoft Entra ID, Active Directory, or the agency's approved recovery store. Microsoft documents firmware upgrades, boot manager changes, and PCR changes as common BitLocker recovery triggers.

## Step 4: Deploy Through a Supported Channel

Microsoft documents supported deployment options for IT-managed Windows devices. Use the path that matches how the fleet is already managed:

- **Microsoft Intune** for cloud-managed or co-managed endpoints.
- **Policy CSP** for MDM-driven policy.
- **Group Policy** for traditional domain-joined estates.
- **Registry keys** for scripted, test, or constrained management scenarios.

For CSP-backed deployment, Microsoft documents `./Device/Vendor/MSFT/Policy/Config/SecureBoot/EnableSecurebootCertificateUpdates` with the enabled value `22852`. For registry-based deployment, Microsoft documents setting `AvailableUpdates` to `0x5944` under `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\SecureBoot`, then monitoring `UEFICA2023Status` under `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\SecureBoot\Servicing`.

Use the Microsoft Support guidance for IT professionals and the current Microsoft Learn troubleshooting article for exact sequencing. Validate each ring against device boot behavior, BitLocker recovery reports, event logs, and `UEFICA2023Status = Updated` before expanding the rollout.

## The Mixed Windows and Linux Angle

Mixed fleets need extra attention because many Linux Secure Boot chains use shim as a first-stage UEFI boot loader. The rhboot shim project describes shim as a first-stage UEFI boot loader that can execute another application and validate it when Secure Boot is enabled. Microsoft documentation states that the Microsoft third-party UEFI CA trust path is used for third-party boot loaders and EFI applications, and the 2023 renewal separates third-party boot loader trust from option ROM trust.

For Linux and dual-boot systems, track two requirements:

1. **Microsoft UEFI CA 2023 must be present in DB** so firmware trusts boot loaders signed under the renewed third-party boot loader CA.
2. **The Linux distribution's Secure Boot chain must be current and supported** so shim, the second-stage boot loader, and related boot components work with the renewed Secure Boot trust path.

Do not assume a Windows deployment policy fully remediates Linux. Windows-managed endpoints, Linux servers, network appliances, and dual-boot workstations may need different update paths and different vendor guidance.

## Why This Matters for Government

State and local agencies operate under continuity-of-operations expectations, budget constraints, and security programs that increasingly depend on strong device trust. NIST SP 800-207 describes zero trust as a model focused on users, assets, and resources rather than implicit network trust. Secure Boot is not a complete zero-trust solution by itself, but it supports the device-integrity foundation that those strategies rely on.

Three practical points matter for CIOs, CISOs, and endpoint teams:

- **This is planned maintenance, not an emergency.** Devices can keep booting and receiving standard Windows updates after the 2011 certificates expire, but they may lose future Secure Boot protections if the 2023 certificates are not installed.
- **BitLocker recovery is the likely user-impact risk.** Firmware and boot-measurement changes can trigger recovery. Escrowed keys and firmware-first sequencing reduce the chance of a help-desk surge.
- **Mixed fleets need two plans.** Windows policy can update Windows-managed devices, but Linux and dual-boot systems require validation of shim, distribution packages, and firmware DB contents.

Build the inventory now, bring firmware current, pilot against varied hardware, and deploy in rings tied to measurable status signals. That sequence keeps the fleet booting safely while preserving Secure Boot protection beyond the 2011 certificate expiration window.

## References

- Windows Secure Boot certificate expiration and CA updates: [https://support.microsoft.com/en-us/topic/windows-secure-boot-certificate-expiration-and-ca-updates-7ff40d33-95dc-4c3c-8725-a9b95457578e](https://support.microsoft.com/en-us/topic/windows-secure-boot-certificate-expiration-and-ca-updates-7ff40d33-95dc-4c3c-8725-a9b95457578e)
- Update Secure Boot Certificates for Windows Devices: [https://learn.microsoft.com/en-us/troubleshoot/windows-client/windows-security/update-secure-boot-certificates](https://learn.microsoft.com/en-us/troubleshoot/windows-client/windows-security/update-secure-boot-certificates)
- Secure Boot Certificate updates: Guidance for IT professionals and organizations: [https://support.microsoft.com/topic/e2b43f9f-b424-42df-bc6a-8476db65ab2f](https://support.microsoft.com/topic/e2b43f9f-b424-42df-bc6a-8476db65ab2f)
- Registry key updates for Secure Boot: [https://support.microsoft.com/en-us/topic/registry-key-updates-for-secure-boot-windows-devices-with-it-managed-updates-a7be69c9-4634-42e1-9ca1-df06f43f360d](https://support.microsoft.com/en-us/topic/registry-key-updates-for-secure-boot-windows-devices-with-it-managed-updates-a7be69c9-4634-42e1-9ca1-df06f43f360d)
- PowerShell: Using the -Decoded parameter in Get-SecureBootUEFI: [https://support.microsoft.com/en-us/topic/powershell-using-the-decoded-parameter-in-get-securebootuefi-fe15ff66-f8c9-445c-b663-59e6084fa824](https://support.microsoft.com/en-us/topic/powershell-using-the-decoded-parameter-in-get-securebootuefi-fe15ff66-f8c9-445c-b663-59e6084fa824)
- Secure Boot DB and DBX variable update events: [https://support.microsoft.com/en-us/topic/kb5016061-secure-boot-db-and-dbx-variable-update-events-37e47cf8-608b-4a87-8175-bdead630eb69](https://support.microsoft.com/en-us/topic/kb5016061-secure-boot-db-and-dbx-variable-update-events-37e47cf8-608b-4a87-8175-bdead630eb69)
- Policy CSP - SecureBoot: [https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-secureboot](https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-secureboot)
- Confirm-SecureBootUEFI: [https://learn.microsoft.com/en-us/powershell/module/secureboot/confirm-securebootuefi](https://learn.microsoft.com/en-us/powershell/module/secureboot/confirm-securebootuefi)
- Get-SecureBootUEFI: [https://learn.microsoft.com/en-us/powershell/module/secureboot/get-securebootuefi](https://learn.microsoft.com/en-us/powershell/module/secureboot/get-securebootuefi)
- BitLocker recovery overview: [https://learn.microsoft.com/en-us/windows/security/operating-system-security/data-protection/bitlocker/recovery-overview](https://learn.microsoft.com/en-us/windows/security/operating-system-security/data-protection/bitlocker/recovery-overview)
- NIST SP 800-207 Zero Trust Architecture: [https://www.nist.gov/publications/zero-trust-architecture](https://www.nist.gov/publications/zero-trust-architecture)
- rhboot shim README: [https://github.com/rhboot/shim/blob/main/README.md](https://github.com/rhboot/shim/blob/main/README.md)
