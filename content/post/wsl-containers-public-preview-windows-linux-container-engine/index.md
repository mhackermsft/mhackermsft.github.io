---
title: 'WSL Containers: Public Preview Brings a Built-In Linux Container Engine to Windows'
date: 2026-07-10T16:27:42+00:00
author: Mike Hacker
tags:
- App Modernization
- Announcements
- How To
- Security
categories:
- App Modernization
summary: Microsoft's WSL container public preview adds a Linux container CLI and native API to WSL, with enterprise management controls that state and local government IT teams can evaluate for managed developer endpoints.
draft: false
image_prompt: A modern Windows developer workstation showing a terminal with WSL container commands, a subtle Linux container motif, and government IT security visuals such as policy shields and managed endpoint icons. Professional Microsoft Azure blog style.
image: cover.png
audio: audio.mp3
---

At Microsoft Build 2026, the Windows Subsystem for Linux (WSL) team introduced WSL containers: a Linux container engine built into WSL for Windows. The public preview was announced on the Windows Command Line blog on [June 29, 2026](https://devblogs.microsoft.com/commandline/wsl-container-is-now-available-for-public-preview/), with general availability targeted for fall 2026. Because this is still preview software, agencies should evaluate it in pilots before treating it as a production dependency.

For state and local government IT teams, the important point is not only developer convenience. It is that container tooling can be managed through familiar Windows enterprise controls, including WSL policy, Intune, group policy, and Microsoft Defender for Endpoint integration paths.

## What WSL Containers Are

The feature has two parts, documented in the [WSL container documentation](https://aka.ms/wslc).

**A container CLI called `wslc.exe`.** When you update to the WSL pre-release that includes the preview, WSL adds `wslc.exe` to the path. The command shape is intentionally familiar for developers who already use container CLIs. The preview also includes `container.exe` as an alias that runs `wslc.exe`.

**A WSL container API.** Windows applications can use Linux containers as part of their own application logic. The API ships in the [`Microsoft.WSL.Containers` NuGet package](https://www.nuget.org/packages/Microsoft.WSL.Containers), with support for C, C++, and C# projections. The package documentation also covers MSBuild and CMake integration, so image build steps can be part of a normal application build.

## Getting Started

Because the feature is in public preview, you opt in through the WSL pre-release channel. From an elevated PowerShell prompt, update WSL:

```powershell
wsl --update --pre-release
```

The official docs show the basic workflow:

```powershell
# Run a one-off container
wslc run --rm -it ubuntu:latest bash -c "echo Hello world from WSL container!"

# List images
wslc image ls

# Run a web server, mapping host port 8080 to container port 80
wslc run -it --rm -d -p 8080:80 --name web nginx

# Hit it from Windows
curl localhost:8080

# Inspect and stop
wslc container ps
wslc container stop web
```

GPU-enabled containers are also part of the preview. The announcement demonstrates CUDA access with this single-line example:

```powershell
wslc run --rm --gpus all pytorch/pytorch:2.5.1-cuda12.4-cudnn9-runtime python -c "import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"
```

## Running Containers From Native Windows Code

The API is the part that opens new Windows application patterns. A native Windows application can pull an image, create a WSL-backed session, start a Linux container, and stream process output without shelling out to a command-line process. The docs organize the C# surface around `WslcService` for prerequisite checks, `Session` for the WSL-backed host, `Container`, and `Process`.

```csharp
using System.Text;
using Microsoft.WSL.Containers;

ComponentFlags missing = WslcService.GetMissingComponents();
if (missing != ComponentFlags.None)
{
    Console.WriteLine($"WSL components are missing ({missing}). Run: wsl --install");
    return;
}

var sessionSettings = new SessionSettings("MyApp", @"C:\WslcData")
{
    CpuCount = 4,
    MemoryMB = 4096
};
var session = new Session(sessionSettings);
session.Start();

var pull = session.PullImageAsync(new PullImageOptions("docker.io/library/alpine:latest"));
await pull;

var initProcess = new ProcessSettings
{
    CmdLine = new[] { "/bin/echo", "Hello from WSL Container!" },
    OutputMode = ProcessOutputMode.Event
};
var containerSettings = new ContainerSettings("alpine:latest")
{
    Name = "hello-container",
    InitProcess = initProcess
};
var container = session.CreateContainer(containerSettings);
container.InitProcess.OutputReceived += data =>
    Console.Write(Encoding.UTF8.GetString(data));
container.Start();
```

Add the package with `dotnet add package Microsoft.WSL.Containers`. The package also includes C++/WinRT support, but the WSL docs explicitly mark that projection as preview and subject to breaking changes.

## Platform Improvements Under the Hood

The preview also updates some WSL plumbing:

- **virtiofs** is the default file system for WSL containers, and the announcement reports roughly 2x faster Windows file access.
- **Consomme** is an experimental networking mode that relays Linux network traffic through Windows. The goal is compatibility with real enterprise networking conditions such as VPNs, proxies, and host security policy.
- **Improved memory reclaim** gradually returns unused memory from the Linux VM back to the Windows host.

The WSL team notes that these lower-level changes are currently enabled for WSL containers and that other WSL-based container tools, including Docker Desktop, Podman Desktop, and Rancher Desktop, can benefit from the platform work.

## Editor Integration

The announcement says VS Code Dev Containers support was added in the `0.462.0-pre-release` build. To use it, open the Dev Containers settings, find **Docker Path**, and set it to `wslc`. Because this support is still in pre-release, government teams should pilot it with a small group before publishing a standard developer workstation baseline.

## Why This Matters for Government

For state and local government, WSL containers are most interesting where development speed meets endpoint governance.

**Procurement and standardization.** A container engine delivered as part of WSL may reduce reliance on separately approved desktop container tooling for some developer scenarios. Treat that as an architecture and operations benefit, not a blanket pricing claim: agencies should still validate Windows, developer-tooling, and support requirements through their normal licensing and procurement process.

**Policy-based control.** The WSL container announcement says administrators can control whether users can use WSL distros, containers, or both, and can define an allowlist for approved container registries. It also says those controls are available through GPO and ADMX policy, with Intune dashboard support expected within weeks of the June 2026 announcement. That aligns with existing [Intune settings for WSL](https://learn.microsoft.com/windows/wsl/intune), which already let administrators manage WSL as a Windows component.

**Endpoint security visibility.** Microsoft documents the [Defender for Endpoint plug-in for WSL](https://learn.microsoft.com/microsoft-365/security/defender-endpoint/mde-plugin-wsl?view=o365-worldwide), and the WSL container announcement says the plug-in is being updated for Linux container events in private preview. For public sector organizations that need continuous endpoint monitoring, this is the right direction: container activity should be visible through the same security operations model used for managed Windows endpoints.

**Networking and proxy compatibility.** The [enterprise WSL guidance](https://learn.microsoft.com/windows/wsl/enterprise) calls out firewall, mirrored networking, DNS tunneling, and proxy considerations for managed environments. WSL containers build on that enterprise story, which matters for agencies with VPNs, web proxies, segmented networks, and compliance-driven workstation policies.

A note on cloud environments: WSL containers run on Windows client endpoints, so the feature itself is not dependent on Azure commercial versus Azure US Government region parity. The images developers pull are still governed by whatever registries and policies the agency allows, which is why registry allowlisting is important for GCC and government-adjacent teams.

## A Community Desktop Experience: WSL Container Desktop

The CLI is useful, but some developers prefer a graphical desktop experience. I built one and am sharing it as an open source community project.

**[WSL Container Desktop](https://github.com/mhackermsft/wslcontainerdesktop)** is a native WinUI 3 and .NET 10 desktop application for managing WSL containers. To be clear: this is an independent community project. It is not a Microsoft product, and it is not affiliated with, endorsed by, or supported by Microsoft. It is published under [GPL-3.0](https://github.com/mhackermsft/wslcontainerdesktop/blob/main/LICENSE). Review the source before running it, and use it at your own risk.

The project README describes these capabilities:

- **Container lifecycle management** including run, start, stop, restart, kill, remove, prune, live logs, an exec terminal, inspect, and live stats.
- **Images, volumes, and networks** with pull, build, tag, push, inspect, and prune.
- **Live performance metrics** for containers, total CPU, and engine health.
- **A built-in Kubernetes manager** that installs a single-node k3s cluster inside WSL and manages nodes, deployments, pods, services, port-forwarding, and YAML apply operations. The k3s project documents single-node installations as fully functional Kubernetes clusters.
- **Registry management**, including an Azure Container Registry flow that uses Azure CLI authentication.

For Azure Container Registry, the app uses the Azure CLI token pattern documented for environments where Docker is not running: `az acr login --name <registryName> --expose-token`. Microsoft documentation says this returns an access token and login server, and that registry access tokens used by `az acr login` are valid for 3 hours. Paired with appropriate Azure RBAC on the registry, that is a better pattern than shared admin passwords for developer workstations.

## Wrapping Up

WSL containers bring a useful preview capability to Windows: a built-in, enterprise-manageable Linux container engine with a familiar CLI, a native API, and controls that fit managed endpoints. It is public preview today, so use it for experimentation and pilots rather than production standards, and watch the WSL release notes as Microsoft works toward general availability in fall 2026.

Start with `wsl --update --pre-release`, read the [official WSL container docs](https://aka.ms/wslc), and if you want a graphical front end to explore with, review [WSL Container Desktop](https://github.com/mhackermsft/wslcontainerdesktop) as a community option.
