# CLAUDE.md — AI Assistant Guide for kali-linux-docker

## Project Overview

This repository provides a fully configurable Kali Linux Docker image with a graphical desktop environment accessible via remote desktop protocols (RDP, VNC, or X2Go). It is intended for security researchers, penetration testers, and CTF participants who want a portable, reproducible Kali Linux environment.

The project was originally created by [onemarcfifty](https://github.com/onemarcfifty/kali-linux-docker) and is maintained in this fork by [canstralian](https://github.com/canstralian/kali-linux-docker).

---

## Repository Structure

```
kali-linux-docker/
├── Dockerfile        # Multi-stage Docker build; conditionally installs desktop/remote access
├── build             # Interactive bash build script; reads env, builds image, creates & starts container
├── env_template      # Template for the env configuration file (committed)
├── .env              # Working configuration file (same content as env_template by default — NOT gitignored)
├── .gitignore        # Ignores: env, .vscode, .DS_Store
├── LICENSE           # Project license
└── README.md         # User-facing documentation
```

> **Note:** The `env` file (no extension) is gitignored. `.env` is **not** gitignored and is committed. Be careful not to commit real credentials.

---

## Key Files Explained

### `Dockerfile`

- Base image: `kalilinux/kali-rolling`
- Accepts build-time `ARG` values passed by the `build` script for all configuration options.
- Installs base packages unconditionally: `sudo`, `wget`, `curl`, `dbus-x11`, `xinit`, `openssh-server`, chosen desktop package, `locales`.
- Generates `/startkali.sh` dynamically at build time — this is the container entrypoint script that starts services on launch.
- Conditionally installs and configures one of three remote access servers based on `REMOTE_ACCESS`:
  - `x2go` — installs `x2goserver`, starts it in `/startkali.sh`
  - `rdp` — installs `xorg`, `xorgxrdp`, `xrdp`; configures port; sets up XFCE session
  - `vnc` — installs `tigervnc-standalone-server`; configures display, users, security, and password rotation loop
- Sets locale to `en_US.UTF-8`.
- Disables the XFCE power manager autostart (avoids blank screen in remote sessions).
- Exposes `SSH_PORT`, `RDP_PORT`, and `VNC_PORT`.
- Entrypoint: `/bin/bash /startkali.sh`

### `build`

- Executable bash script run with `sudo ./build`.
- Sources the `env` file at the top: `export $(grep -v '^#' env | xargs)`.
- Sets defaults for any unset port/display variables.
- Presents interactive menus for: Desktop Environment, Remote Access, Kali Package, Network, Build Architecture — only if not already set in `env`.
- Prompts for: Docker image name, container name, host/container mount paths, username, password (hidden input).
- Displays a confirmation summary before building.
- Runs `docker image build` with all `--build-arg` values.
- Runs `docker create` with network and port mapping.
- Runs `docker start`.
- Cleans up by unsetting all imported env vars.

### `env_template` / `.env`

These files use shell variable syntax and are sourced (not `.env` dotenv format). Variables:

| Variable | Default | Description |
|---|---|---|
| `XRDP_PORT` | `13389` | Host port for RDP |
| `XVNC_DISPLAY` | `8` | VNC display number |
| `XVNC_PORT` | `5908` | Host port for VNC |
| `XSSH_PORT` | `20022` | Host port for SSH / X2Go |
| `XDESKTOP_ENVIRONMENT` | `xfce` | Desktop: `xfce`, `mate`, `kde`, `e17`, `gnome`, `i3`, `i3-gaps`, `live`, `lxde` |
| `XREMOTE_ACCESS` | `rdp` | Remote protocol: `vnc`, `rdp`, `x2go` |
| `XKALI_PKG` | `core` | Kali package set: `arm`, `core`, `default`, `everything`, `firmware`, `headless`, `labs`, `large`, `nethunter` |
| `XNETWORK` | `bridge` | Docker network: `bridge` or `host` |
| `XBUILD_PLATFORM` | `amd64` | Build arch: `amd64` or `arm64` |
| `DOCKERIMG` | `custom/kali-linux` | Local Docker image tag |
| `CONTAINER` | `kali-linux` | Docker container name |
| `HOSTDIR` | `/home/user` | Host path to volume-mount into container |
| `CONTAINERDIR` | `/opt` | Container path for the volume mount |
| `USERNAME` | `user` | Non-root user created inside container |
| `PASSWORD` | `password123` | Password for container user |

---

## Development Workflow

### Setup

```bash
git clone <repo-url>
cd kali-linux-docker
cp env_template env        # create local env (gitignored)
# edit env with your desired values
```

### Building the Container

```bash
sudo ./build               # interactive if env is incomplete
```

The script builds the Docker image, creates the container, then starts it.

### Connecting to the Container

| Protocol | Default Port | Notes |
|---|---|---|
| SSH | `20022` | `ssh -p 20022 <USERNAME>@localhost` |
| X2Go | `20022` | X2Go client; uses same SSH port |
| RDP | `13389` | Any RDP client; works reliably only with XFCE |
| VNC | `5908` | VNC viewer; display `:8` |

### Rebuilding

If you change Dockerfile or env settings, re-run `sudo ./build`. You may need to stop and remove the old container first:

```bash
docker stop kali-linux
docker rm kali-linux
docker rmi custom/kali-linux
sudo ./build
```

---

## Known Constraints & Conventions

1. **RDP only works with XFCE desktop** — do not combine `XREMOTE_ACCESS=rdp` with other desktop environments.
2. **The `env` file (no extension) is gitignored**, `.env` is not — never commit real passwords.
3. **`/startkali.sh` is generated at build time** — it does not exist in the repo; it is assembled by `RUN echo ...` statements in the Dockerfile.
4. **Build script requires `sudo`** — Docker daemon access requires root or the user to be in the `docker` group.
5. **`XVNC_PORT` is derived** from `XVNC_DISPLAY` as `590${XVNC_DISPLAY}` — keep them consistent.
6. **The `build` script sources `env` using `export $(grep -v '^#' env | xargs)`** — values must not contain spaces or special characters that break `xargs` parsing.
7. **Base image is `kalilinux/kali-rolling`** — expect frequent upstream changes; pin the digest if reproducibility is required.

---

## AI Assistant Guidelines

When assisting with this repository:

- **Do not modify credentials** in `.env` or `env_template` without explicit user instruction.
- **Do not commit the `env` file** (gitignored), but `.env` (not gitignored) may be committed — exercise caution with its contents.
- **Security context:** This is a legitimate penetration testing and CTF tooling project. Changes to the Dockerfile that install security tools are expected and appropriate.
- **Dockerfile changes:** Any modification to what gets installed must account for the conditional build logic (`RUN if [ "x..." = "x${REMOTE_ACCESS}" ]`). Preserve that pattern.
- **Port changes:** If changing default ports, update `env_template`, `.env`, and the Dockerfile `EXPOSE` line consistently.
- **No test suite exists** — validation is done by building and launching the container. There are no unit tests to run.
- **Shell compatibility:** The `build` script uses bash features (`(( ))` arithmetic, arrays). Do not convert to POSIX `sh`.
- **Avoid adding complexity** — this is an intentionally simple, single-file Docker project. Do not introduce `docker-compose`, Makefiles, or CI/CD pipelines unless explicitly requested.

---

## Branch Convention

- Active development branch: `claude/add-claude-documentation-V0Qu2`
- Upstream default: `main`
- Local default: `master`
