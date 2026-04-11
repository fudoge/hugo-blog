---
title: "Replacing Obsidian Sync with a Self-Hosted LiveSync Stack"
slug: obsidian-sync-via-self-hosted-livesync-with-tailsclae-caddy
description: "How I built my own Obsidian sync setup using CouchDB, Caddy, and Tailscale inside a Proxmox LXC container, with notes on client setup and off-site backup."
date: 2026-04-11T02:26:09+09:00
image: thumb.png
math: 
license: 
hidden: false
comments: true
draft: false

tags:
    - homelab
categories:
    - homelab

---

## 🧋 Introduction

### My Journey with Note-Taking Apps

I use Obsidian as my persoanl knoweldge base instead of Notion.
The reason is simple:
- Notion feels too heavy
- It comes with too many features I do not really need
- The loading speed can feel slow at times

Obsidian, on the other hand, feels lightweight, fast, and much closer to the kind of workflow I want for managing my notes

### Sync options for Obsidian

There are many possible options, and I spend quite a bit of time looking through them.
- Official options
  - **Obsidian Sync**
    - It's polished and reliable, but it is a paid service.
  - **iCloud**
    - This can be a solid choice if you are fully in the Apple ecosystem.
    - However, it becomes less practical when you also use non-Apple devices. (I use NixOS btw)
- Unofficial options
  - **Git & GitHub**
    - I tried this before, but the mobile experience was not great, expecially when the repository became larger.
    - On iPhone, using apps like Working Copy for pull and push operations did not feel very reliable.
  - **Syncthing**
    - Peer-to-peer sync is a very practical approach, expecially when combined with a WireGuard-based VPN like Tailscale.
    - The main problem for me was the lack of official support on iPhone.
  - **⭐ Self-hosted livesync(with CouchDB)**
    - The option stood out the most to me.

### Why I Chose Self-Hosted LiveSync

In the end, I decided to go with self-hosted LiveSync, backed by CouchDB, for a few reasons:
- I already have my own home server running on Proxmox
- The LiveSync plugin has a strong community and over 10k GitHub start
- It looked fun to build and self-host

In this post, I will walk through how I set up my Obsidian sync stack using CouchDB + Caddy + Tailscale inside a Proxmox LXC container.

---
## 🏗️ Provisioning an LXC Container

First, We will provision an **LXC Container** in a **Proxmox** Cluster. \
I use the **Terraform** provider [bgp/proxmox](https://registry.terraform.io/providers/bpg/proxmox/latest), to manage resources declaratively.

### Downloading Debian 13 LXC Image

Although we are using Terraform, the `proxmox_download_file` resource currently downloads the LXC image repeatedly, because of a checksum verification issue. \
So for now, I will download manually instead.

First, connect to your proxmox cluster:
```bash
ssh <your-proxmox-cluster>
```

Then, list officially supported LXC images:
```bash
pveam available
```

Download the Debian13 image:
```bash
pveam download local debian-13-standard_13.1-2_amd64.tar.zst
```

### Terraform Module

We will create a module to make the code reusable.

```hcl
# ./modules/lxc_container/main.tf
terraform {
  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "0.101.0"
    }
  }
}

resource "proxmox_virtual_environment_container" "container" {
  description = var.description

  node_name = var.node_name
  vm_id     = var.vm_id

  unprivileged = true
  # If true, it cannot be destoryed.
  protection   = var.protection

  features {
    nesting = true
  }

  # Required to turn on tailscaled on LXC container.
  device_passthrough {
    path = "/dev/net/tun"
  }

  # If you want you can make resource quotas to variables.
  cpu {
    cores = 2
    limit = 2
    units = 1024
  }

  memory {
    dedicated = 2048
    swap      = 512
  }

  initialization {
    hostname = var.hostname

    ip_config {
      ipv4 {
        address = "dhcp"
      }
    }

    user_account {
      keys     = var.ssh_keys
      password = random_password.container.result
    }
  }

  wait_for_ip {
    ipv4 = true
  }

  network_interface {
    name = var.network_interface
  }

  disk {
    datastore_id = var.datastore_id
    size         = var.disk_size
  }

  operating_system {
    template_file_id = var.template_file_id
    type             = var.os_type
  }

  startup {
    order      = "3"
    up_delay   = "60"
    down_delay = "60"
  }

  dynamic "mount_point" {
    for_each = var.mount_points

    content {
      path   = mount_point.value.path
      volume = mount_point.value.volume

      size          = try(mount_point.value.size, null)
      read_only     = try(mount_point.value.read_only, null)
      backup        = try(mount_point.value.backup, null)
      replicate     = try(mount_point.value.replicate, null)
      shared        = try(mount_point.value.shared, null)
      quota         = try(mount_point.value.quota, null)
      acl           = try(mount_point.value.acl, null)
      mount_options = try(mount_point.value.mount_options, null)
    }
  }

  tags = var.tags
}

resource "random_password" "container" {
  length           = 16
  override_special = "_%@"
  special          = true
}

output "container_password" {
  value     = random_password.container.result
  sensitive = true
}
```

Here's the module's `variables.tf`.
```hcl
# ./module/lxc_container/variables.tf

variable "description" {
  type        = string
  description = "LXC description"
}

variable "node_name" {
  type        = string
  description = "Proxmox node name"
}

variable "vm_id" {
  type        = number
  description = "LXC VM ID"
}

variable "protection" {
  type        = bool
  description = "Whether to prevent data"
}

variable "hostname" {
  type        = string
  description = "Container hostname"
}

variable "ssh_keys" {
  type        = list(string)
  description = "SSH public keys for root account"
  default     = []
}

variable "network_interface" {
  type        = string
  description = "Container network interface name"
}

variable "datastore_id" {
  type        = string
  description = "Datastore ID for root disk"
}

variable "disk_size" {
  type        = number
  description = "Root disk size in GB"
}

variable "template_file_id" {
  type        = string
  description = "OS template file ID"
}

variable "os_type" {
  type        = string
  description = "Container OS type"

  validation {
    condition = contains([
      "alpine",
      "centos",
      "debian",
      "fedora",
      "ubuntu",
      "unmanaged"
    ], var.os_type)
    error_message = "os_type must be one of the supported Proxmox LXC OS types."
  }
}

variable "mount_points" {
  description = "Additional mount points for the container"
  type = list(object({
    path          = string
    volume        = string
    size          = optional(string)
    read_only     = optional(bool)
    backup        = optional(bool)
    replicate     = optional(bool)
    shared        = optional(bool)
    quota         = optional(bool)
    acl           = optional(bool)
    mount_options = optional(list(string))
  }))
  default = []
}

variable "tags" {
  description = "Tags"
  type        = list(string)
}
```

If we expose the IPv4 address as an output, we can connect to the LXC container more easily:
```hcl
# ./modules/lxc_container/outputs.tf
output "ipv4" {
  value = proxmox_virtual_environment_container.container.ipv4
}
```

### `provider.tf` 

To passthrough a device, you need to access Proxmox using an administrator account (`root@pam`) with a password:
```hcl
terraform {
  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "0.101.0"
    }
  }
}

provider "proxmox" {
  endpoint = <your_endpoint>
  username = var.proxmox_username
  password = var.proxmox_password
  insecure = true

  ssh {
    agent    = true
    username = "root"
  }
}
```

### `main.tf`
```hcl
# main.tf
locals {
  # Import your SSH public keys here
  ssh_keys = [
  ]
}

module "couchdb" {
  source       = "./modules/lxc_container"
  node_name    = <your-node-name>
  description  = "CouchDB LXC"
  datastore_id = <your-datastore-id>
  protection   = true # Recommended

  vm_id             = 500 # You can change if you want
  network_interface = <your-network-interface>
  disk_size         = 32 # You can change if you want
  hostname          = "couchdb" # You can change if you want

  # File ID format: `datasotre_id:type/<image>`
  template_file_id = "local:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst"
  os_type          = "debian"
  ssh_keys         = local.ssh_keys

  # Example tags
  tags = ["debian", "couchdb", "obsidian"]
}

output "couchdb_ipv4" {
  value = module.couchdb.ipv4
}
```

### `variables.tf`

```variables.tf
# varialbes.tf
# `terraform.tfvars` should be ignored by Git.
# or, you can use environment variable: `export TF_VAR_<variable_name>`
variable "proxmox_username" {
  type    = string
  default = "root@pam"
}

variable "proxmox_password" {
  type      = string
  sensitive = true
}
```

### Applying & Accessing

Now, you can create the LXC Container!

```bash
terraform init

terraform plan

terraform apply -auto-approve
```

After applying, You can access container now:
```bash
ssh root@<your-container-ip>
```

---
## 🪏 Settingup CouchDB + LiveSync

### Download CouchDB

First, update the package manager and upgrade installed packages. \
Then, install the dependencies required to follow the CouchDB installation instructions:
```bash
apt update
apt upgrade -y
apt install -y gpg curl sudo
```

Next, follow the official CouchDB installation guide [here](https://docs.couchdb.org/en/stable/install/unix.html#installation-using-the-apache-couchdb-convenience-binary-packages).


During installation, a TUI prompt will appear to configure CouchDB. \
You can use the following options:
- type: `standalone`
- bind address: `127.0.0.1`
- Erlang magic cookie: `<your-random-long-string>`
- admin password: `<your-admin-password>`

Store the informations somewhere private and secure.

To verify that CouchDB is running, check its service status:
```bash
systemctl status couchdb
```

### Install Obsidian LiveSync

Replace the placeholders and run the script below:
```bash
curl -s https://raw.githubusercontent.com/vrtmrz/obsidian-livesync/main/utils/couchdb/couchdb-init.sh | hostname=http://<YOUR SERVER IP>:5984 username=<INSERT USERNAME HERE> password=<INSERT PASSWORD HERE> bash
```

You can also refer to the latest instructions [here](https://github.com/vrtmrz/obsidian-livesync/blob/main/docs/setup_own_server.md#2-run-couchdb-initsh-for-initialise).

If the output looks like this:
```bash
-- Configuring CouchDB by REST APIs... -->
{"ok":true}
""
""
""
""
""
""
""
""
""
<-- Configuring CouchDB by REST APIs Done!
```
then the database has been initialized successfully.


---
## 🔒 Setting up Tailscale

**Tailscale** is a peer-to-peer VPN built on the top of WireGuard, so it lets you securely connect you devices from anywhere. 

### Install Tailscale on Your devices

Log in to Tailscale and open the dashboard.

The dashboard already provides clear installation instructions for each platform, so you can simply follow them. \
You have to install Tailscale in all the devices you want to use Obsidian.
![Tailscale](tailscale.png)

Once Tailscale is installed and connected, you can also verify it from inside your LXC container. \
A new network interface should appear:
![IP link show after installation](tailscale-lxc.png)

### Set Up MagicDNS

Tailscale also supports internal DNS and HTTPS certificates support issued by Let's Encrypt. \

In Dashboard -> DNS, enable MagicDNS and HTTPS Certificates.


---
## 👮 Setup Caddy

**Caddy** is a simple web server that can also act as a reverse-proxy. \
It integrates especially well with Tailscale, which makes it a great fit fot this setup 

You can install Caddy [here](https://caddyserver.com/docs/install#debian-ubuntu-raspbian).

In `/etc/caddy/Caddyfile`, configure the reverse proxy:
```Caddyfile
# /etc/caddy/Caddyfile
# Match your tailnet MagicDNS FQDN
https://<hostname>.<your-tailnet>.ts.net {
	reverse_proxy 127.0.0.1:5984 {
		header_up Host {host}
		header_up X-Forwarded-For {remote_host}
		header_up X-Forwarded-Proto {scheme}
	}
}
```

In `/opt/couchdb/etc/local.ini`, configure the CORS options.
```ini
[chttpd]
bind_address = 127.0.0.1
port = 5984
enable_cors = true

[cors]
credentials = true
origins = app://obsidian.md, capacitor://localhost, http://localhost
headers = accept, authorization, content-type, origin, referer
methods = GET, PUT, POST, HEAD, DELETE, OPTIONS
max_age = 3600
```

In `/etc/default/tailscaled`, make sure the `caddy` user can access Tailscale in order to manage HTTPS certificates.
```bash
TS_PERMIT_CERT_UID=caddy
```

Then, restart the services
```bash
systemctl restart tailscaled
systemctl restart caddy
systemctl restart couchdb
```

---
## 🍰 Client usage

Almost done! \
The only things left are migrating your existing vault and connecting your client devices.

### Generate a Setup URI

The commands below do not need to be run inside the LXC container. \
You can run them on your own laptop instead.

```bash
export hostname=<your-hostname>
export database=obsidiannotes # Change this if you want
export passphrase=<random-passphrase>
export username=<username>
export password=<password>

deno run -A https://raw.githubusercontent.com/vrtmrz/obsidian-livesync/main/utils/flyio/generate_setupuri.ts

# Generated Passphrase & Link:
Your passphrase of Setup-URI is:  <passphrase>
This passphrase is never shown again, so please note it in a safe place.
obsidian://setuplivesync?settings=....
```

### Migrate Your Existing Vault

Follow these steps:
1. Install the **Self-hosted LiveSync** plugin in Obsidian.
2. Enable the plugin
3. For Device Setup Method, select Use a Setup URI.
   - Enter your passphrase and URI(`obsidian://setuplivesync?settings=...`)
4. Select **"I am setting up a new server for the first time / I want to reset my existing server."**
5. Follow the remaining steps.
6. Reload the app once without saving.


### Register your other devices

I created a new valut in my phone, and followed these steps:
1. Install the **Self-hosted LiveSync** plugin in Obsidian.
2. Enable the Plugin
3. For Device Setup Method, select a Setup URI. (QR code didn't work for me)
   - Enter with your passphrase and URI(`obsidian://setuplivesync?settings=...`)
4. Select **"My remote server is already set up. I want to join this device"**
5. Select **"This Vault is empty, or contains only new files that are not on the server"**
6. Follow the remaining setup steps.
7. Reload the app once without saving.

### Settings
I am using LiveSync presets, for immediate syncing. \
In most case, the sync latency is under one second.

---
## ☁️ Off-Site Backup

Because the database can become **a single point of failure**, you should be prepared for outages or storage failures. \
With an off-site backup, you can restore your data more safely.

### Git & GitHub

It may feel a little awkward to keep using Git here, but it is still a practical option for a secondary backup because it is lightweight enough for frequent pushes. \
You don't need to install it on every device. \
Instead, you can install a Git plugin on your main desktop or laptop and push changes continuously. \
Auto-push is also supported by Git plugins. \
In this setup, the `.git` folder is automatically ignored when syncing via LiveSync, so you don't need to worry about sync conflicts or Git history growth affecting LiveSync.

### Cloud Object Storage

Object storage services such as **Cloudflare R2** or **AWS S3** are highly reliable, and their lifecycle policies are very useful. \
Periodically backing up your data to object storage can be a good option. \
You can also automate backups using `cron` and the AWS CLI.

---
## 📚 References

- [GitHub - vrtmrz/obsidian-livesync](https://github.com/vrtmrz/obsidian-livesync/blob/main/docs/setup_own_server.md#c-install-couchdb-directly)
- [Terraform Registry - bgp/proxmox](https://registry.terraform.io/providers/bpg/proxmox/latest/docs/resources/virtual_environment_container)
- [Techstuff - Proxmox - LXC template and containers - Getting Started](https://techstuff.leighonline.net/2024/01/10/proxmox-lxc-containers/)
- [Source Code(GitHub) - LXC module](https://github.com/fudoge/home-infra/blob/main/terraform/10.proxmox-lxc/modules/lxc_container/main.tf)
- [CouchDB - Installation](https://docs.couchdb.org/en/stable/install/unix.html#installation-using-the-apache-couchdb-convenience-binary-packages)
- [Tailscale - Tailscale in LXC Containers](https://tailscale.com/docs/features/containers/lxc/lxc-unprivileged)
- [Caddy - Installation](https://caddyserver.com/docs/install)
- [Caddy Certificates of Tailscale](https://tailscale.com/docs/integrations/web-servers/caddy/caddy-certificates)
