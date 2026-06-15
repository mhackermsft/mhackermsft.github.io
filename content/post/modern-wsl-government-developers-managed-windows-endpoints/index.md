---
title: 'Modern WSL for Government Developers: Running Linux AI and Container Workloads on Managed Windows Endpoints'
date: 2026-06-15T18:58:59+00:00
author: Mike Hacker
tags:
- App Modernization
- AI
- Security
- How To
categories:
- App Modernization
summary: A technical deep-dive on how modern Windows Subsystem for Linux supports GPU-accelerated AI, container, and Linux workloads on managed Windows endpoints for government development teams.
draft: false
image_prompt: A managed Windows government developer workstation running WSL Linux, containers, and local AI GPU workloads, with security policy overlays for Intune, Defender, and firewall controls.
image: cover.png
audio: audio.mp3
---

Government development teams live in a split world. Their endpoints are managed, hardened Windows machines governed by Intune and Microsoft Defender for Endpoint, but the workloads they build increasingly assume Linux: containers, Python AI pipelines, open-source build chains, and Linux-native service tooling. In the past, that often meant dual booting, standing up a separate VM, or sending test data to a cloud sandbox. Windows Subsystem for Linux (WSL) reduces that friction by letting developers run a real Linux kernel, systemd services, GPU-accelerated model inference, and container workloads on the same governed Windows endpoint.

A quick clarification: there is no WSL 3 product. WSL is serviced through the Microsoft Store or GitHub release packages, WSL 2 is the current default architecture, and the WSL repository is public under an MIT license. As of June 15, 2026, the current release on the [microsoft/WSL releases page](https://github.com/microsoft/WSL/releases) is **2.7.8**, posted June 6, 2026, with the Linux kernel updated to **6.18.33.1-1**. The meaningful enterprise changes are landing as WSL 2 incremental releases and Windows integration features: mirrored networking, DNS tunneling, Hyper-V firewall integration, the Microsoft Defender for Endpoint plug-in for WSL, and Intune policy controls.

## The architecture in one paragraph

WSL 2 runs a real Linux kernel inside a lightweight utility virtual machine managed by Windows. Microsoft describes WSL 2 distributions as isolated containers inside that managed VM. Each distribution keeps its own Linux user space, packages, file system, services, and configuration, while WSL manages the shared kernel and virtualization boundary. Because WSL 2 uses a genuine Linux kernel rather than a system-call translation layer, developers get the compatibility needed by Docker, systemd-based services, and GPU compute frameworks.

Configuration is split across two files. Global VM settings live in `%UserProfile%\.wslconfig` under a `[wsl2]` section. Per-distribution settings live in `/etc/wsl.conf` inside each distribution. Use `.wslconfig` for VM-level choices such as networking behavior, firewall behavior, memory, and processors. Use `wsl.conf` for distribution behavior such as systemd, the default user, mounts, interoperability, and GPU access.

## systemd support changes deployment

Current Ubuntu installations created by `wsl --install` use systemd by default, but existing distributions and other Linux distributions may still need it enabled explicitly. To enable systemd in a distribution, edit `/etc/wsl.conf`:

```ini
# /etc/wsl.conf
[boot]
systemd=true
```

Then restart WSL from PowerShell:

```powershell
wsl --shutdown
```

After the distribution starts again, confirm systemd-managed services:

```bash
systemctl list-unit-files --type=service
```

This matters because many Linux tools assume systemd is available. For government teams standardizing on reproducible development environments, it means local Linux service definitions can more closely match the unit files used in production Linux environments.

## GPU and CUDA passthrough for local model inference

WSL exposes the Windows GPU to Linux through para-virtualization. The per-distribution GPU setting is enabled by default, but it can be made explicit in `/etc/wsl.conf`:

```ini
# /etc/wsl.conf
[gpu]
enabled=true
```

Install the NVIDIA CUDA-enabled Windows driver for WSL on the Windows host. Do not install a Linux display driver inside the WSL distribution. Microsoft documents CUDA support in WSL for PyTorch, TensorFlow, Docker, and NVIDIA Container Toolkit workflows. Microsoft lists WSL kernel version 5.10.43.3 or higher for its CUDA-on-WSL guidance, and NVIDIA recommends using the latest WSL kernel. The WSL 2.7.8 release is above those thresholds.

A practical local check inside Ubuntu on WSL looks like this:

```bash
cat /proc/version               # verify the WSL kernel version
nvidia-smi                      # confirm the GPU is visible in Linux

python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu130
python -c 'import torch; print(torch.cuda.is_available())'
```

PyTorch packaging changes over time. The May 13, 2026 PyTorch 2.12.0 release notes state that CUDA 13.0 is the stable default and CUDA 12.6 remains available for older driver baselines. If your agency pins GPU drivers, use the PyTorch install selector or previous-versions page rather than guessing a CUDA wheel URL.

For containerized inference, install and configure the NVIDIA Container Toolkit for the container engine you use. NVIDIA documents this sample Docker validation command after the toolkit and GPU driver are installed:

```bash
sudo docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi
```

The governance upside is significant: a developer can prototype against a small open-weight model on the local device, reducing the need to move sensitive test data to an external service during early experimentation. That does not replace the agency data-classification process, but it can keep early exploration inside the managed endpoint boundary while the formal review is completed.

## Mirrored networking helps with VPN and proxy friction

The default WSL networking mode is NAT. NAT is workable, but it can create friction with split DNS, VPN routes, and internal endpoints. On Windows 11 22H2 and higher, mirrored networking is available through `.wslconfig`:

```ini
# %UserProfile%\.wslconfig
[wsl2]
networkingMode=mirrored
dnsTunneling=true
autoProxy=true
firewall=true
```

Microsoft's WSL networking guidance documents the benefits of mirrored mode: IPv6 support, connecting from Linux to Windows services by using the localhost address 127.0.0.1, multicast support, direct LAN reachability for WSL, and improved VPN compatibility. The same guidance documents `dnsTunneling=true` as a way to answer DNS requests through a virtualization channel rather than a network packet, which helps in complex VPN environments. `autoProxy=true` makes WSL use Windows HTTP proxy information, so Linux traffic can follow the agency's existing proxy path.

The April 25, 2026 WSL 2.7.3 release notes included route-mirroring fixes and a boot-time check for IPv6 disabled through the registry in mirrored mode.

If you need to expose a WSL service to the LAN under mirrored mode, use Hyper-V firewall rules instead of disabling the firewall. First confirm the WSL VMCreatorId if you want to verify the value on a device:

```powershell
Get-NetFirewallHyperVVMCreator
```

Then create the inbound rule from an elevated PowerShell session:

```powershell
New-NetFirewallHyperVRule -Name DevApi -DisplayName 'Dev API' `
  -Direction Inbound -VMCreatorId '{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}' `
  -Protocol TCP -LocalPorts 8080
```

## Enterprise configuration, Intune, and governance

The enterprise story is what makes WSL viable for a regulated endpoint fleet. Microsoft's enterprise setup guidance calls for Windows 10 22H2 or higher, or Windows 11 22H2 or higher, with WSL 2.0.9 or higher. Advanced networking features require Windows 11 22H2 or higher.

- **Microsoft Defender for Endpoint plug-in for WSL.** The plug-in applies to Microsoft Defender for Endpoint Plan 2 and gives security teams visibility into activity from running WSL distributions. Microsoft documents important limitations: the plug-in does not provide every Defender for Endpoint capability inside WSL, such as antimalware, threat and vulnerability management, or response commands for the WSL logical device. It also is not supported on ARM64 processors, multi-session Windows variants, or custom kernel and custom kernel command-line configurations.
- **Intune-managed WSL controls.** Intune can manage WSL as a Windows component through the Windows settings catalog. The documented settings include allowing or blocking WSL, blocking WSL 1, blocking the inbox WSL version, blocking the debug shell, blocking passthrough disk mount, and controlling whether users can configure sensitive `.wslconfig` settings such as custom kernels, custom networking, firewall configuration, nested virtualization, and kernel debugging. If you want a standard `.wslconfig` file, distribute it through your normal device-configuration channel and use Intune policy to restrict user overrides where appropriate.
- **Hyper-V firewall integration.** Starting with Windows 11 22H2 and WSL 2.0.9 or higher, Windows firewall rules automatically apply to WSL by default. That helps extend host egress controls to Linux workloads without a separate firewall policy inside each distribution.
- **Approved golden images.** Build a hardened distribution once, then distribute it with WSL import and export commands:

```powershell
wsl --export <Distro> <FileName>
wsl --import <Distro> <InstallLocation> <FileName>
```

One security property is worth underlining for risk reviewers: root inside a WSL distribution is not Windows administrator. When a Linux binary accesses a Windows file, it does so with the permissions of the Windows user who launched `wsl.exe`. Running Bash has the same Windows-side permissions as running PowerShell as that user.

Be clear about the current gaps. Microsoft lists several unsupported enterprise controls: using Windows tools to patch Linux user space, using Windows Update to update WSL distribution contents, controlling which distributions users can access, and centrally controlling root inside the distribution. Plan for Linux patching and configuration management through a Linux configuration manager, scheduled package updates, or rebuilt approved images.

## Why This Matters for Government

State and local agencies are under pressure to modernize applications and evaluate AI while maintaining strict endpoint-security mandates. WSL lets a small development team do real Linux engineering on the same Intune-managed, Defender-protected laptop the agency already issues, rather than procuring separate hardware or weakening device policy.

GPU passthrough enables local model prototyping that can keep sensitive test data on the device during early experimentation. That can support the practical review work agencies perform for CJIS, IRS Publication 1075, HIPAA, StateRAMP, FedRAMP, and other control frameworks before a cloud architecture is approved. Microsoft documents Azure compliance offerings for [CJIS](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-cjis), [IRS Publication 1075](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-irs-1075), [HIPAA](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-hipaa-us), [StateRAMP](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-stateramp), and [FedRAMP](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-fedramp); WSL does not replace those cloud compliance obligations, but it gives development teams a governed endpoint pattern for pre-cloud engineering work.

Mirrored networking, DNS tunneling, proxy inheritance, and Hyper-V firewall integration address the VPN and outbound-inspection friction that often appears on government networks. Because WSL is a Windows client capability rather than an Azure regional service, teams do not need to design around Azure commercial versus Azure Government region availability for the local development environment itself.

The result is not an unmanaged escape hatch. With Microsoft Defender for Endpoint Plan 2, Intune policy, Hyper-V firewall controls, and approved images, WSL can become a governed Linux development capability that fits the endpoint-management model agencies already use.

## Getting started

1. Update WSL from the Microsoft Store or current GitHub release package, then verify with `wsl --version`. As of June 15, 2026, the current release is 2.7.8.
2. Install a glibc-based distribution such as Ubuntu or Debian. If systemd is not already enabled, set `systemd=true` in `/etc/wsl.conf` and restart with `wsl --shutdown`.
3. On Windows 11 22H2 or higher, evaluate `networkingMode=mirrored`, `dnsTunneling=true`, `autoProxy=true`, and `firewall=true` in `.wslconfig` for your agency network.
4. For AI workloads, install the NVIDIA CUDA-enabled driver for WSL on the host and validate GPU visibility with `nvidia-smi` inside the distribution.
5. If your licensing and platform requirements are met, deploy the Defender for Endpoint plug-in for WSL and build an approved golden image with `wsl --export`.

A managed Windows endpoint can now serve as a governed Linux AI and container development workstation. For government teams that have had to choose between developer productivity and endpoint control, modern WSL makes that trade-off easier to manage.
