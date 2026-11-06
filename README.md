# Unreal Devops Pipeline

**Automated nightly builds, playtime analytics and bug reporting for an Unreal Engine 5 team project.**
Perforce + TeamCity + a Windows build agent, self-hosted on Proxmox, with a Discord notification at the end of every build.

> Context : School project (HEAJ, 2025), built for a narrative walking-simulator game developed by a student team ([Itch.io](https://antoniogozzi.itch.io/drop-out))

<img width="1860" height="384" alt="image12" src="https://github.com/user-attachments/assets/2cb0307d-502b-45ec-9e2c-a139e74c9bb4" />

## Table of contents
 
- [The problem](#the-problem)
- [The solution](#the-solution)
  - [1. Why TeamCity and not Jenkins or GitLab CI](#1-why-teamcity-and-not-jenkins-or-gitlab-ci)
  - [2. How it works](#2-how-it-works)
  - [3. Build steps](#3-build-steps)
  - [4. Infrastructure](#4-infrastructure)
- [Strengths and limitations](#strengths-and-limitations)

## The problem
 
**Slow, manual builds.** Packaging an Unreal project is long and blocks the developer's machine. Doing it by hand during a sprint wastes time, and testers did not always have the latest version.

## The solution

**CI/CD pipeline**: Syncs from Perforce, compiles code and shaders, cooks and packages the game, zips and uploads it, then notifies the team (TeamCity, Perforce, PowerShell, Docker, Discord webhook)
 
### 1. Why TeamCity and not Jenkins or GitLab CI
 
TeamCity is JetBrains' CI platform and has good Perforce integration. The goal was Windows builds and a pipeline that could be reused for a larger final project. Perforce and TeamCity are also common in game studios.

### 2. How it works

1. **Trigger**: a scheduled nightly build, or a developer submit to Perforce. If the agent is already busy, the new job is queued and replaced when newer submits arrive.
2. **Sync**: the Windows build agent uses the P4 CLI to pull the latest changes.
3. **Build**: Unreal Automation Tool compiles, cooks and stages the project. A parameter selects Development or Shipping.
4. **Archive**: the packaged game is zipped with a fixed naming scheme and moved to the storage VM, served online behind a password-protected reverse proxy.
5. **Notify**: a Discord webhook posts success or failure with a link to the build.

<img width="629" height="179" alt="image9" src="https://github.com/user-attachments/assets/496941a3-d443-4d76-89f6-2f3a6bfdb6a3" />

### 3. Build steps
 
Each step is a PowerShell command in TeamCity:
 
| Step | Purpose |
|---|---|
| Generate Project Files | Sets parameters and generates the Unreal project files |
| Package Game | Runs `BuildCookRun` (build, cook, stage, archive) |
| Zip Game | Compresses the output with `Compress-Archive` |
| Publish | Moves the zip to the storage location |

### 4. Infrastructure
 
Everything is self-hosted on one **Proxmox** host, split into VMs. Access goes through a **WireGuard VPN** and an **Nginx reverse proxy**.
 
| VM | Specs | Role |
|---|---|---|
| `databases` | Linux, 2 cores, 8 Gb RAM, 200 Gb SSD | MySQL and Postgres (TeamCity uses an external Postgres, as JetBrains recommends for production) |
| `web01` | Linux, 8 cores, 20 Gb RAM, 400 Gb SSD | Perforce server, plus Docker/Portainer running **TeamCity Server**, **OpenProject** and **n8n** |
| `build01` | Windows 10, 3 cores, 7 Gb, 144 Gb SSD | Build agent: Unreal Engine, Visual Studio, P4V / P4 CLI, TeamCity Agent |
| `storage01` | Linux, 1 core, 6 gb RAM, 500 Gb SSD | Public and private storage for archived builds |

<img width="888" height="676" alt="image8" src="https://github.com/user-attachments/assets/6d811b47-6b60-4bf2-8c35-1af41cce6741" />

## Strengths and limitations
 
**Strengths**
- **Industry-standard stack**: Perforce and TeamCity scale to bigger projects and teams.
- **Autonomy**: nightly builds mean testers have a fresh version every morning with no manual work.
- **Easy to reuse**: pipelines, jobs and steps can be cloned for a new project, and parameters isolate project-specific values.
  
**Limitations (and what I'd do next)**
- **No automated tests.** The pipeline proves the game builds, not that it works. Next step: unit and functional tests in the pipeline.
- **No build retention.** Zips accumulate on the storage VM. Next step: a cleanup step that deletes archives older than a set age.
- **No pipeline health check.** 500 GB is allocated for builds, but nothing warns when it fills up. Next step: a step before packaging that checks free space, warns on Discord under 10 % and aborts under 5 %.
- **Single point of failure.** Everything runs on one Proxmox host. Next step: dedicated servers and an off-site backup.
- **Analytics data loss on crash.** The CSV is written only at the end of a session. Next step: append to the file on every event.
- **Feedback latency.** A Windows build takes time between a submit and the notification. The commands are already optimized to avoid wiping and re-downloading the depot, and only changed assets and code are recompiled.
