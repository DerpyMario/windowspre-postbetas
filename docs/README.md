# Setup Guide 📖

How to set up this project in **Docker** and on **GitHub**.

This repository is a fork of [dockur/windows](https://github.com/dockur/windows) that focuses on running
pre-release and post-release Windows builds (Insider Preview / beta ISOs) inside a container. The
[main readme](../readme.md) covers day-to-day usage and the FAQ, and [environment.md](environment.md) lists every
supported environment variable. This page covers the two things those pages do not: getting the container running
end to end on your own machine, and wiring up the GitHub side of the repository (Actions, registries, Codespaces).

**Contents**

- [Part 1 — Docker](#docker)
  - [Requirements](#requirements)
  - [Step 1: Verify that KVM works](#step-1-verify-that-kvm-works)
  - [Step 2: Start the container](#step-2-start-the-container)
  - [Step 3: Connect to Windows](#step-3-connect-to-windows)
  - [Building the image from this repository](#building-the-image-from-this-repository)
  - [Installing a beta or Insider Preview build](#installing-a-beta-or-insider-preview-build)
  - [Mount points](#mount-points)
  - [Ports](#ports)
  - [Common settings](#common-settings)
  - [Troubleshooting](#troubleshooting)
- [Part 2 — GitHub](#github)
  - [Step 1: Fork or clone](#step-1-fork-or-clone)
  - [Step 2: Repository variables](#step-2-repository-variables)
  - [Step 3: Repository secrets](#step-3-repository-secrets)
  - [Step 4: Enable Actions and package writes](#step-4-enable-actions-and-package-writes)
  - [Workflow reference](#workflow-reference)
  - [Publishing an image with the Build workflow](#publishing-an-image-with-the-build-workflow)
  - [Installation and validation tests](#installation-and-validation-tests)
  - [Docker Hub description sync](#docker-hub-description-sync)
  - [Download mirror checks](#download-mirror-checks)
  - [Codespaces and devcontainers](#codespaces-and-devcontainers)
  - [Dependency updates](#dependency-updates)
  - [Running the CI checks locally](#running-the-ci-checks-locally)
  - [Branches](#branches)
- [Quick reference](#quick-reference)

---

<a id="docker"></a>

# Part 1 — Docker 🐳

## Requirements

| Requirement | Notes |
|---|---|
| Docker or Podman | On a Linux host with KVM support. |
| Docker Desktop or Podman Desktop | Only on Windows 11, with nested virtualization enabled. |
| RAM | At least 2 GB free, more for a usable desktop. `RAM_SIZE` defaults to `4G`. |
| Disk | At least 32 GB free. `DISK_SIZE` defaults to `64G`, and the image is sparse, so it grows over time. |
| CPU | Hardware virtualization (`Intel VT-x` / `AMD-V`) enabled in the BIOS or UEFI. |

> [!NOTE]
> Docker Desktop on Linux, macOS, and Windows 10 does not pass `/dev/kvm` into containers, so it is not supported.
> Without KVM the VM either refuses to start or falls back to software emulation that is too slow to install Windows.

## Step 1: Verify that KVM works

Do this before anything else — almost every "it never boots" report comes down to a missing KVM device.

```bash
sudo apt install cpu-checker
sudo kvm-ok
```

A working host reports:

```text
KVM acceleration can be used
```

Then confirm the device node exists and that your user can reach it:

```bash
ls -l /dev/kvm
```

If it is missing, check that virtualization is enabled in the BIOS/UEFI, that nested virtualization is enabled when
the host is itself a VM, and that your VPS or cloud provider allows it. If `kvm-ok` passes but the container still
says KVM is unavailable, temporarily add `privileged: true` to rule out a permission problem with the device.

## Step 2: Start the container

### Docker Compose (recommended)

Create a `compose.yml`:

```yaml
services:
  windows:
    image: dockurr/windows
    container_name: windows
    environment:
      VERSION: "11"
      RAM_SIZE: "8G"
      CPU_CORES: "4"
      DISK_SIZE: "128G"
    devices:
      - /dev/kvm
      - /dev/net/tun
    cap_add:
      - NET_ADMIN
    ports:
      - 8006:8006
      - 3389:3389/tcp
      - 3389:3389/udp
    volumes:
      - ./windows:/storage
    restart: always
    stop_grace_period: 2m
```

Then run:

```bash
docker compose up -d      # start
docker compose logs -f    # watch the installation
docker compose down       # stop
```

`stop_grace_period: 2m` matters: it gives Windows time to shut down cleanly instead of being killed mid-write.

### Docker CLI

```bash
docker run -it --rm --name windows \
  -e "VERSION=11" \
  -p 8006:8006 \
  --device=/dev/kvm \
  --device=/dev/net/tun \
  --cap-add NET_ADMIN \
  -v "${PWD:-.}/windows:/storage" \
  --stop-timeout 120 \
  docker.io/dockurr/windows
```

### Podman

The same flags work with `podman run`. Run it rootful (or give your user access to `/dev/kvm` and `/dev/net/tun`),
and add `--cap-add NET_ADMIN`.

### Kubernetes

A manifest is included in the repository root:

```bash
kubectl apply -f kubernetes.yml
```

It creates a 64 Gi `PersistentVolumeClaim`, a single-replica `Deployment` with the `Recreate` strategy (never run
two replicas against one disk), and a `ClusterIP` `Service` exposing 8006, 3389/TCP, 3389/UDP, and 5900. The pod
mounts `/dev/kvm` and `/dev/net/tun` from the host, so schedule it onto a node where those exist.

## Step 3: Connect to Windows

- **Web viewer** — open <http://127.0.0.1:8006/>. Installation is fully automatic; sit and watch until the desktop appears.
- **RDP** — connect to the container's IP with any Microsoft Remote Desktop client. Default credentials are
  username `Docker`, password `admin` (change them with `USERNAME` and `PASSWORD`).

The web viewer is intended for the installation phase. It is less responsive than RDP and has no clipboard sharing,
so switch to RDP once the desktop is up.

## Building the image from this repository

The published images come from the `Dockerfile` in the repository root. To build it yourself:

```bash
git clone https://github.com/DerpyMario/windowspre-postbetas.git
cd windowspre-postbetas
docker build -t windows:local .
```

Build arguments you can override with `--build-arg`:

| Argument | Default | Purpose |
|---|---|---|
| `VERSION_ARG` | `latest` | Written to `/etc/version` inside the image and used as the arm64 base image tag. |
| `VERSION_VIRTIO` | `1.9.60` | VirtIO driver package bundled as `/var/drivers.txz`. |
| `VERSION_WSDD` | `1.27` | `wsdd-native` release used for network discovery. |
| `VERSION_UDF` | `1.2.0` | `udfread` release used to read UDF ISO images. |
| `VERSION_BLINTER` | `1.0.112` | Batch-file linter installed via pip. |

The build is multi-arch: on `amd64` it starts from `qemux/qemu` and installs the tooling, while on `arm64` it uses
`dockurr/windows-arm` as the base. For both architectures at once:

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t windows:local .
```

To run your local build from Compose, replace `image:` with a build context:

```yaml
services:
  windows:
    build: .
    container_name: windows
    # ...rest of the config unchanged
```

> [!TIP]
> `.dockerignore` already excludes `.git`, `.github`, `.devcontainer`, the compose files, and all `*.md` files, so
> documentation changes never invalidate the build cache.

## Installing a beta or Insider Preview build

This is the part specific to this fork. There are two ways to install a build that is not in the `VERSION` table.

### Option A: download from a URL

Point `VERSION` at any ISO URL and the container downloads it on first start:

```yaml
environment:
  VERSION: "https://software.download.prss.microsoft.com/dbazure/Windows11_InsiderPreview_..."
```

> [!IMPORTANT]
> Microsoft's Insider Preview download links are signed and time-limited — they carry `t=`, `P1=`, `P2=` query
> parameters and stop working after a few hours. If the download fails with an authorization or expiry error, fetch a
> fresh link from the [Windows Insider Preview downloads](https://www.microsoft.com/software-download/windowsinsiderpreviewiso)
> page and update `VERSION`. Because the link is long and contains `&` and `%` characters, always keep it quoted in
> your compose file. The `compose.yml` in this repository is a worked example of this setup.

### Option B: use a local ISO

Skip the download entirely by binding an ISO into the container. `VERSION` is ignored when a custom image is present:

```yaml
volumes:
  - ./Windows11_InsiderPreview.iso:/custom.iso
```

The container looks for `custom.iso` or `boot.iso` at the container root, and also inside the storage folder, so
dropping the file into your `./windows` storage directory as `custom.iso` works too.

> [!TIP]
> Keep one storage folder per build. Switching `VERSION` against an existing disk does not reinstall Windows — the
> installer only runs when the storage folder has no system image yet.

### Unattended answers and post-install steps

Beta builds sometimes need a nudge that the stock answer file does not provide:

```yaml
environment:
  COMMAND: 'reg add "HKLM\Software\Example" /v Enabled /t REG_DWORD /d 1 /f'
volumes:
  - ./oem:/oem
```

Anything in the folder bound to `/oem` is copied to `C:\OEM`, and an `install.bat` inside it runs during the final
step of the installation — the usual place to install drivers or enable the Insider channel.

## Mount points

| Container path | Purpose |
|---|---|
| `/storage` | **Required in practice.** The virtual disk and downloaded ISO. Without it, everything is lost when the container is removed. |
| `/storage2`, `/storage3`, … | Extra disks, paired with `DISK2_SIZE`, `DISK3_SIZE`, and so on. |
| `/custom.iso`, `/boot.iso` | A local ISO to install instead of downloading one. |
| `/shared` | Host folder exposed to Windows as the `Shared` desktop folder and drive `Z:`. |
| `/data` | Fallback for `/shared`, used when `/shared` is not bound. This is what `./winc:/data` in the repository's `compose.yml` does. |
| `/shared2`, `/data2`, … | Additional shares, exposed as `Data2` and upward. |
| `/oem` | Copied to `C:\OEM`; its `install.bat` runs at the end of installation. |
| `/dev/kvm` | Hardware acceleration. Required. |
| `/dev/net/tun` | Networking. Required. |
| `/dev/vhost-net` | Only needed when `DHCP: "Y"` is set. |

## Ports

| Port | Protocol | Purpose |
|---|---|---|
| 8006 | TCP | Web viewer (noVNC-based). |
| 3389 | TCP + UDP | RDP. Map both protocols — UDP carries the video stream. |
| 5900 | TCP | Raw VNC. Not exposed by the Dockerfile; add it yourself if you want a native VNC client. |

The `Dockerfile` declares `EXPOSE 3389 8006` and `VOLUME /storage`.

## Common settings

The image defaults are `VERSION=11`, `RAM_SIZE=4G`, `CPU_CORES=2`, and `DISK_SIZE=64G`. The settings you are most
likely to change:

| Variable | Example | Effect |
|---|---|---|
| `VERSION` | `11`, `10l`, `2025`, or an ISO URL | Which Windows to install. |
| `RAM_SIZE` | `8G`, `half`, `max` | Memory given to the VM. |
| `CPU_CORES` | `4`, `half`, `max` | Virtual cores. |
| `DISK_SIZE` | `256G` | Disk capacity. Can be increased later without data loss, but you must extend the partition inside Windows. |
| `USERNAME` / `PASSWORD` | `bill` / `gates` | Account created during installation. |
| `LANGUAGE` | `French` | Installation language. |
| `REGION` / `KEYBOARD` | `en-US` | Locale and keyboard layout, when they differ from `LANGUAGE`. |
| `AUDIO` | `Y` | Streams audio to the web viewer (then enable **Audio** under **Settings → Advanced**). |
| `DHCP` | `Y` | Lets Windows get its own IP from your router, on a macvlan network. |
| `ARGUMENTS` | `-device usb-host,vendorid=0x1234,productid=0x1234` | Extra QEMU arguments, used for USB passthrough. |

See [environment.md](environment.md) for the complete list, grouped by category.

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| Container exits immediately, log mentions KVM | `/dev/kvm` is not passed through, or the host has no virtualization. Re-run [step 1](#step-1-verify-that-kvm-works). |
| Web viewer never leaves the boot screen | Wait — the ISO download and installation can take a long time. Follow `docker compose logs -f` to see actual progress. |
| Download fails with an authorization or 403 error | An expired Insider Preview link. Get a fresh URL, or switch to a local ISO. |
| Installation fails on a fresh start with a USB drive attached | Detach USB mass storage until Windows Setup finishes — it can be picked as the system disk and formatted. |
| Windows reinstalls or the disk seems empty after restart | No `/storage` bind mount, so the disk lived in the container layer. |
| Windows starts but the network is dead | `NET_ADMIN` capability or `/dev/net/tun` missing. |
| Out of disk space partway through | The virtual disk grows on demand; free space on the host or lower `DISK_SIZE`. |

---

<a id="github"></a>

# Part 2 — GitHub 🐙

The `.github` folder ships a complete CI/CD setup: build and publish, linting, PR review, live installation tests,
Docker Hub description sync, and download-mirror checks. On a fresh fork none of it works until you configure the
variables and secrets below.

> [!IMPORTANT]
> **Every workflow is manual.** They all run on `workflow_dispatch` only — nothing fires on a push, a pull request, a
> release, or a schedule. Start each one from **Actions → _(workflow)_ → Run workflow**. The two reusable workflows,
> `settings.yml` and `install.yml`, are the exception: they run only when another workflow calls them, so they never
> trigger on their own either.

This means pull requests get no automatic linting or review, the Docker Hub description no longer syncs by itself,
and the download mirrors are not checked on a schedule. Run **Test**, **Update**, and **Links** by hand when you
want those.

## Step 1: Fork or clone

```bash
gh repo fork DerpyMario/windowspre-postbetas --clone
# or
git clone https://github.com/DerpyMario/windowspre-postbetas.git
```

Then open **Settings → Actions → General** and make sure workflows are allowed to run. Forks start with Actions
disabled until you enable them from the **Actions** tab.

## Step 2: Repository variables

Set these under **Settings → Secrets and variables → Actions → Variables**. The Build workflow reads them, and a
missing value produces a broken image tag rather than a clear error.

| Variable | Example | Used for |
|---|---|---|
| `MAJOR` | `1` | Major part of the image tag. |
| `MINOR` | `0` | Minor part. Auto-incremented after each successful build. |
| `NAME` | `Windows` | Value of the `org.opencontainers.image.title` label. |

Each build publishes `latest` plus `MAJOR.MINOR`, and then bumps `MINOR` for the next run.

## Step 3: Repository secrets

Under **Settings → Secrets and variables → Actions → Secrets**:

| Secret | Required for | What to put in it |
|---|---|---|
| `DOCKERHUB_USERNAME` | Build, Update | Your Docker Hub username. |
| `DOCKERHUB_TOKEN` | Build, Update | A Docker Hub access token with read/write scope (**not** your password). |
| `DOCKERHUB_REPO` | Build, Update | Target repository, for example `yourname/windows`. |
| `REPO_ACCESS_TOKEN` | Build | A personal access token used to create the release and increment `MINOR`. Needs contents write and Actions variables write on this repository. |
| `GITLAB_URL`, `GITLAB_USERNAME`, `GITLAB_TOKEN` | Build (mirror step) | Optional GitLab mirror. |
| `MAILTO`, `MAIL_CONNECTION` | Build, Links | Optional notification mail. `MAIL_CONNECTION` is an SMTP connection URL. |

`GITHUB_TOKEN` is provided automatically — do not create it yourself.

> [!TIP]
> Everything except the Docker Hub trio is optional. If you only want images in GHCR, you can delete the Docker Hub
> login step and the `${{ secrets.DOCKERHUB_REPO }}` line from the image list in `.github/workflows/build.yml`, and
> drop `hub.yml` entirely. Steps whose secrets are empty fail the job rather than skipping it.

## Step 4: Enable Actions and package writes

The Build job requests `packages: write`, `actions: write`, and `contents: read`. For pushes to GHCR to succeed:

1. **Settings → Actions → General → Workflow permissions** — allow read and write, or at minimum leave the
   workflow-declared permissions in effect.
2. After the first successful push, the package appears at `ghcr.io/<owner>/<repo>` and is private by default.
   Change its visibility under the package's own settings if you want it public.

## Workflow reference

| Workflow | File | Trigger | What it does |
|---|---|---|---|
| Build | `build.yml` | Manual | Runs the checks, builds `linux/amd64` + `linux/arm64`, pushes to Docker Hub and GHCR, creates a release, bumps `MINOR`, mirrors to GitLab, sends mail. |
| Check | `check.yml` | Manual, or called by **Build** and **Test** | ShellCheck over `src`, hadolint on the `Dockerfile`, XML validation of `assets`, and JSON/YAML validation. |
| Test | `test.yml` | Manual | Thin wrapper that runs **Check**. |
| Review | `review.yml` | Manual | Shared review workflow from `action-pack/.github`. Needs `pull-requests: write` and `checks: write`. |
| Update | `hub.yml` | Manual | Pushes `readme.md` to Docker Hub as the repository description. |
| Links | `links.yml` | Manual | Validates every download mirror with aria2 and mails on failure. |
| Validation | `validate.yml` | Manual | Matrix of real installation tests across desktop, server, legacy, win9x, and tiny editions. |
| Settings | `settings.yml` | Called by **Validation** | Applies shared test parameters (timeouts, reboot limits) and calls **Installation**. |
| Installation | `install.yml` | Called by **Settings** | Builds the image and installs Windows on a runner, then asserts caption, edition, build number, and platform. |

`settings.yml` and `install.yml` keep only `workflow_call` because `workflow_dispatch` allows at most 10 inputs and
does not support the `number` type — `install.yml` declares 22 inputs, several of them numeric. Run **Validation**
to reach them, or call **Installation** from a workflow of your own as shown [below](#installation-and-validation-tests).

> [!NOTE]
> **Review** was written for pull requests and reads the PR from the event payload. Dispatched manually there is no
> pull request in context, so expect it to find nothing to review. It is kept on `workflow_dispatch` for
> consistency; put it back on `pull_request` if you want it to do its job, or delete it.

## Publishing an image with the Build workflow

1. Confirm `MAJOR`, `MINOR`, `NAME`, and the Docker Hub secrets are set.
2. Go to **Actions → Build → Run workflow**.
3. The `shellcheck` job runs first; the build only starts if it passes.
4. On success the image is pushed to both registries and a `v<MAJOR>.<MINOR>` release is created.

Pull the result with:

```bash
docker pull ghcr.io/derpymario/windowspre-postbetas:latest
# or from Docker Hub, using whatever DOCKERHUB_REPO points at
docker pull yourname/windows:latest
```

To use it in your compose file, swap the image line:

```yaml
services:
  windows:
    image: ghcr.io/derpymario/windowspre-postbetas:latest
```

The build is concurrency-limited to one run at a time (`group: build`, `cancel-in-progress: false`), so a second
dispatch queues behind the first instead of cancelling it.

## Installation and validation tests

`validate.yml` runs the real thing: it builds the image on a runner, boots Windows, waits for the installation to
finish, and checks what actually landed. Each matrix entry declares an `expected_caption`, an optional
`expected_edition`, a `minimum_build`, and a `platform`.

Useful things to know before you run it:

- **It needs KVM on the runner.** The job's first real step is `test -c /dev/kvm`, so it fails fast on any runner
  without the device. It targets `ubuntu-24.04` by default.
- **It is slow.** Each job has a 180-minute timeout, with a 9000-second installation timeout and an 18000-second
  reboot timeout inherited from `settings.yml`.
- **It frees disk space first**, deleting the preinstalled Android SDK, Node modules, and similar, because the
  runner disk is otherwise too small for a Windows install.
- **It builds from a branch**, controlled by the `branch` input (default `dev`). If that branch does not exist, the
  workflow falls back to `master` and says so in the log.
- **A public noVNC tunnel is available** via the `enable_tunnel` input when you call `install.yml` directly, which is
  the only practical way to watch a failing installation. `settings.yml` sets it to `false`.

To test a single version without the full matrix, call **Installation** from your own workflow:

```yaml
jobs:
  try-beta:
    uses: ./.github/workflows/install.yml
    with:
      name: Windows 11 Insider
      version: "https://example.com/Windows11_InsiderPreview.iso"
      branch: master
      expected_caption: Windows 11
      minimum_build: 22000
      platform: x64
      enable_tunnel: true
      tunnel_delay: 1800
      kill_on_failure: true
      install_timeout: 9000
      reboot_timeout: 18000
      minimum_reboots: 1
      boot_loop_limit: 10
```

## Docker Hub description sync

`hub.yml` pushes `./readme.md` to Docker Hub as the repository description, using the repository's GitHub
description as the short description. It needs the same three Docker Hub secrets. Since it is manual, remember to
run it after editing the readme. It reads `readme.md` specifically, so changes to this guide under `docs/` are never
reflected on Docker Hub.

## Download mirror checks

`links.yml` validates every download mirror referenced by the source scripts using aria2, then mails `MAILTO` when a
mirror breaks. This is the early-warning system for "the download suddenly 404s" reports. It only needs `MAILTO` and
`MAIL_CONNECTION`; without them the mail step fails while the validation results still show in the job log.

It used to run on a daily schedule. Now that it is manual, run it before a release, or re-add a `schedule:` trigger
if you want the mirrors watched unattended:

```yaml
on:
  workflow_dispatch:
  schedule:
    - cron: '0 0 * * *'
```

## Codespaces and devcontainers

The `.devcontainer` folder holds one config per Windows version, so you can pick the edition when creating a
Codespace instead of editing files:

```text
.devcontainer/
├── devcontainer.json              # Windows 11 Pro (the default)
├── codespaces.yml                 # the Compose file all configs share
├── 010 - Windows 11 Enterprise/
├── 020 - Windows 11 LTSC/
├── 030 - Windows 10 Pro/
└── …
```

Each `devcontainer.json` sets `VERSION` through `containerEnv`, forwards port 8006 with a notification, and points
at the shared `codespaces.yml`. That Compose file uses `RAM_SIZE: half`, `CPU_CORES: max`, and `DISK_SIZE: max` so
the VM scales to whatever machine type you picked, and runs `docker system prune --all --force` as its
`initializeCommand` to reclaim space before starting.

To add a version of your own, copy an existing folder, rename it (the numeric prefix controls the order in the
picker), and change `name` and `VERSION`:

```json
{
  "name": "Windows 11 Insider",
  "service": "windows",
  "containerEnv": {
    "VERSION": "https://example.com/Windows11_InsiderPreview.iso"
  },
  "forwardPorts": [8006],
  "dockerComposeFile": "../codespaces.yml",
  "initializeCommand": "docker system prune --all --force"
}
```

> [!IMPORTANT]
> `codespaces.yml` pulls `ghcr.io/dockur/windows` — the upstream image. Change it to
> `ghcr.io/derpymario/windowspre-postbetas` if you want Codespaces to run your own build, and make sure the package
> is public or that the Codespace can authenticate to GHCR.

> [!NOTE]
> The Codespace machine must expose `/dev/kvm`. If the container reports that KVM is unavailable, the machine type
> does not support nested virtualization and there is no workaround from inside the Codespace. A 4-core machine is
> the realistic minimum, and the largest available disk is worth choosing since the Windows disk grows on demand.

## Dependency updates

Both bots are configured, so pick one and disable the other to avoid duplicate PRs:

- **Dependabot** (`.github/dependabot.yml`) — weekly checks of Docker base images and GitHub Actions, with a 7-day
  cooldown. Action updates are limited to majors; minor and patch bumps are ignored.
- **Renovate** (`.github/renovate.json`) — `config:recommended` with the dependency dashboard disabled. Requires the
  Renovate app to be installed on the repository.

## Running the CI checks locally

The same checks `check.yml` runs, before you push:

```bash
# Shell scripts (matching the CI options)
shellcheck -x --source-path=src -e SC1091 -e SC2034 -e SC2153 -e SC2317 src/*.sh

# Dockerfile
docker run --rm -i hadolint/hadolint hadolint --ignore DL3006 --ignore DL3008 --ignore DL3067 - < Dockerfile

# Answer files
for f in assets/*.xml; do xmllint --noout "$f" || echo "invalid: $f"; done
```

Note that hadolint runs with `failure-threshold: warning` in CI, so warnings fail the build, not just errors.

## Branches

`master` is the default branch, and `beta-pre-releases` tracks beta and pre-release work.

Because every workflow is manual, the branch you pick in the **Run workflow** dropdown is the branch whose workflow
file and code are used. One extra wrinkle applies to the installation tests: `install.yml` builds from its own
`branch` input (default `dev`) rather than the dispatched ref, and silently falls back to `master` when that branch
does not exist — so pass `branch: beta-pre-releases` explicitly when you want to test that branch.

---

<a id="quick-reference"></a>

# Quick reference 🚀

```bash
# Run it
docker compose up -d && docker compose logs -f

# Web viewer                      http://127.0.0.1:8006
# RDP                             <container-ip>:3389, user Docker, password admin

# Build the image locally
docker build -t windows:local .

# Multi-arch build
docker buildx build --platform linux/amd64,linux/arm64 -t windows:local .

# Pull a published build
docker pull ghcr.io/derpymario/windowspre-postbetas:latest

# Deploy to Kubernetes
kubectl apply -f kubernetes.yml

# Lint like CI does
shellcheck -x --source-path=src -e SC1091 -e SC2034 -e SC2153 -e SC2317 src/*.sh
```

| Task | Where |
|---|---|
| Every environment variable | [docs/environment.md](environment.md) |
| Usage and FAQ | [readme.md](../readme.md) |
| Kubernetes manifest | [kubernetes.yml](../kubernetes.yml) |
| Compose example with a beta ISO | [compose.yml](../compose.yml) |
| CI workflows | [.github/workflows](../.github/workflows) |
