# Homelab

Infrastructure experiments

## Current Stack
```
┌─────────────────────────────────────┐
│ Applications (Coolify, Gitea, etc.) │ 
├─────────────────────────────────────┤
│ Traefik (Reverse Proxy)             │ 
├─────────────────────────────────────┤
│ Ubuntu (OS)                         │ 
├─────────────────────────────────────┤
│ Proxmox (Virtualization)            │ 
└─────────────────────────────────────┘
```
## Hardware

| Component | Specification |
|-----------|---------------|
| **Processor** | Intel® Core™ i7-1060NG7 (10th Gen, 4C/8T) |
| **Base Speed** | 1.2 GHz (Burst: 3.8 GHz) |
| **Graphics** | Intel® Iris® Plus Graphics |
| **Memory** | 16GB LPDDR4 |
| **Storage** | 1TB NVMe SSD |

## Setup

### Proxmox VE 9.2
- Download ISO and creat bootable USB
- Installed on bare metal

### Ubuntu LXC Container
Created using [community script](https://community-scripts.org/scripts/ubuntu?from=scripts#install):
- 1 CPU core
- 512MB RAM
- 2GB disk

### Traefik

**Installation:**
```bash
# Download latest version (check for newer releases)
TRAEFIK_VERSION="3.7.5"
ARCH=$(dpkg --print-architecture 2>/dev/null || uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/')

wget "https://github.com/traefik/traefik/releases/download/v${TRAEFIK_VERSION}/traefik_v${TRAEFIK_VERSION}_linux_${ARCH}.tar.gz"

# Install binary
sudo tar -xzf "traefik_v${TRAEFIK_VERSION}_linux_${ARCH}.tar.gz" -C /usr/local/bin

# Verify
traefik version
```

**Configuration:**

In Traefik can refer to two different things:

- 1) Install configuration/static configuration (startup): set up connections to **providers** and define the **entrypoints** Traefik will listen to (these elements don't change often).

- 2) Routing configuration/dynamic configuration: contains everything that defines how the requests are handled by your system. This configuration can change and is seamlessly hot-reloaded, without any request interruption or connection loss.

Traefik gets its routing configuration from providers: whether an orchestrator, a service registry, or a plain old configuration file.

1) **Install configuration**: There are three different, mutually exclusive (i.e. you can use only one at the same time), ways to define install configuration options in Traefik (in order):

    - 1) In a [configuration file](https://doc.traefik.io/traefik/getting-started/configuration-overview/#configuration-file)
    - 2) In the [command-line arguments](https://doc.traefik.io/traefik/getting-started/configuration-overview/#configuration-file)
    - 3) As [environment variables](https://doc.traefik.io/traefik/getting-started/configuration-overview/#configuration-file)


2) **Providing Dynamic (Routing) Configuration to Traefik**: defines how Traefik routes incoming requests to the correct services Depending on your environment and preferences, there are several ways to supply this routing configuration (checks [docs](https://doc.traefik.io/traefik/reference/routing-configuration/dynamic-configuration-methods/#providing-dynamic-routing-configuration-to-traefik)):
    - Use TOML or YAML files.
    - Docker and ECS Providers: Use container labels.
    - Kubernetes Providers: Use annotations.
    - KV Providers : Use key-value pairs.
    - Other Providers (Consul, Nomad, etc.) : Use tags.

  