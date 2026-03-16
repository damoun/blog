---
layout: single
title: Running Talos Linux on Hetzner Cloud
categories: [ops]
tags: [talos, hetzner, terraform, opentofu, kubernetes]
---

I've run Kubernetes in all kinds of setups over the years: GKE, AKS, EKS, Rancher, RKE2, k3s on Raspberry Pis. For my lab on Hetzner Cloud, I wanted something I wouldn't have to babysit, something I could manage entirely through code, and honestly, something fun to play with. [Talos Linux](https://www.talos.dev/) checked all those boxes. I use it to run my side projects and personal services.

If you haven't heard of it: Talos is a Linux distribution built only for running Kubernetes. There's no SSH, no shell, no package manager. The whole OS is read-only and you talk to it through a gRPC API with `talosctl`. It sounds extreme, but once you try it, going back to managing a "real" OS underneath Kubernetes feels like unnecessary work. The [official Hetzner installation guide](https://docs.siderolabs.com/talos/v1.12/platform-specific-installations/cloud-platforms/hetzner#hetzner) is a good starting point.

**Prerequisites:** Packer, Terraform/OpenTofu, `talosctl`, and a Hetzner Cloud account with an [API token](https://docs.hetzner.com/cloud/api/getting-started/generating-api-token/).
{: .notice--info}

## Why I picked Talos

With most Kubernetes setups, there's always this underlying OS to deal with: patching it, hardening it, rotating SSH keys, hoping nothing drifts. Talos just removes all of that. The attack surface is tiny, upgrades are atomic, and every node looks exactly the same because they're all driven from the same machine config.

Hetzner doesn't have a managed Kubernetes offering that fits what I want. So I'm building my own cluster, and I'd rather spend my time on what runs on Kubernetes than on the nodes themselves. Talos lets me do that.

## How it works: Packer + Terraform/OpenTofu

Hetzner doesn't ship Talos images, so you have to bring your own. My workflow has three steps:

1. **Build a Talos snapshot** with Packer
2. **Spin up the infrastructure** with Terraform/OpenTofu
3. **Bootstrap the cluster** with `talosctl`

Everything lives in [damoun/terraform-hcloud-talos](https://github.com/damoun/terraform-hcloud-talos), a Terraform/OpenTofu module.

### Step 1: Build the snapshot with Packer

Hetzner doesn't let you upload raw disk images directly, so you need a workaround. The trick is to boot a throwaway server in Hetzner's rescue mode, which gives raw access to `/dev/sda`, then write the Talos image straight to disk. Packer automates this and captures the result as a labeled snapshot.

Talos has an [Image Factory](https://factory.talos.dev/) where you pick the extensions you need (I use it for WireGuard and storage drivers) and get a custom image URL with a unique schema ID.

```hcl
variable "talos_version" {
  default = "v1.12.4"
}

local {
  image_url = "https://factory.talos.dev/image/ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515/${var.talos_version}/hcloud-amd64.raw.xz"
}

source "hcloud" "talos" {
  location      = "fsn1"
  server_type   = "cx23"
  image         = "debian-12"
  rescue        = "linux64"
  ssh_username  = "root"
  snapshot_labels = {
    type    = "talos"
    version = var.talos_version
  }
  snapshot_name = "talos-${var.talos_version}"
}

build {
  sources = ["source.hcloud.talos"]

  provisioner "shell" {
    inline = [
      "apt-get install -y wget xz-utils",
      "wget -O /tmp/talos.raw.xz ${local.image_url}",
      "xz -d /tmp/talos.raw.xz",
      "dd if=/tmp/talos.raw of=/dev/sda bs=4M",
      "sync",
    ]
  }
}
```

```bash
packer build -var="hcloud_token=$HCLOUD_TOKEN" packer/talos.pkr.hcl
```

You run this once per Talos version. The snapshot gets labeled `type=talos,version=v1.12.4` so Terraform/OpenTofu can find it later.

**Tip:** That long hash in the factory URL is a schema ID that pins which extensions are baked into your image. You can generate your own at [factory.talos.dev](https://factory.talos.dev/).
{: .notice--success}

### Step 2: Provision with Terraform/OpenTofu

This module only handles the Hetzner infrastructure: servers, networking, firewall, and an optional load balancer for the Kubernetes API on port 6443. Talos bootstrapping is a separate step.

Terraform/OpenTofu looks up the snapshot by its labels and provisions the cluster:

```hcl
data "hcloud_image" "talos" {
  with_selector     = "type=talos,version=${var.talos_version}"
  with_architecture = "x86"
  most_recent       = true
}

resource "hcloud_network" "cluster" {
  name     = var.cluster_name
  ip_range = var.network_cidr
}

resource "hcloud_firewall" "cluster" {
  name = var.cluster_name

  rule {
    description = "Talos apid"
    direction   = "in"
    protocol    = "tcp"
    port        = "50000"
    source_ips  = var.admin_cidrs
  }

  rule {
    description = "ICMP"
    direction   = "in"
    protocol    = "icmp"
    source_ips  = ["0.0.0.0/0", "::/0"]
  }
}

resource "hcloud_server" "control_plane" {
  count       = var.control_plane_count
  name        = "${var.cluster_name}-cp-${count.index}"
  server_type = var.server_type
  location    = var.location
  image       = data.hcloud_image.talos.id
  # ...
}
```

The module outputs control plane and worker IPs (public, private, and IPv6), which you feed into the bootstrap step.

### Step 3: Bootstrap with talosctl

Once the servers are up, you generate machine configs and apply them:

```bash
talosctl gen config my-cluster https://<control-plane-ip>:6443
talosctl apply-config --nodes <control-plane-ip> --file controlplane.yaml
talosctl bootstrap --nodes <control-plane-ip>
talosctl kubeconfig --nodes <control-plane-ip>
```

After bootstrap completes, you have a running cluster and a kubeconfig ready to use with `kubectl`.

**Note:** There's also a [Talos Terraform/OpenTofu provider](https://github.com/siderolabs/terraform-provider-talos) that can handle this step as code. It's still early stage, but I use it in my own setup to keep everything declarative.
{: .notice--info}

## What it costs

| Component           | Qty | Monthly (excl. VAT)  |
| ------------------- | --- | -------------------- |
| CX23 control planes | 3   | 3 x 3.99 = **11.97** |
| CX23 workers        | 3   | 3 x 3.99 = **11.97** |
| LB11 load balancer  | 1   | **7.49**             |
| Talos snapshot      | 1   | **~0.01** (~203 MB)  |
| **Total**           |     | **~31.44**           |

Prices in EUR after the [April 2026 adjustment](https://docs.hetzner.com/general/infrastructure-and-availability/price-adjustment/).

## Things I learned along the way

**Version pinning matters.** The Packer snapshot version and the Talos installer version used during bootstrap must match. If they don't, the node boots fine but fails to join the cluster because the installer expects a different config schema. I keep one `talos_version` variable referenced in both Packer and Terraform/OpenTofu to avoid this.
{: .notice--danger}

**You'll probably need Image Factory.** Talos doesn't support arbitrary kernel modules out of the box. If you need WireGuard or storage drivers, you have to bake them into a custom image. That's why my Packer template points to a factory URL instead of a vanilla release.

**The firewall can stay tiny.** Only port 50000 from my IPs and ICMP. Kubernetes API access goes through the load balancer or direct to the control plane nodes.

## Wrapping up

Once everything is running, operating Talos feels remarkably calm. Upgrades are just `talosctl upgrade`. You watch the node drain, reboot, and rejoin the cluster. No SSH, no scripts, no crossing fingers. For a setup I want to enjoy, not maintain, that's exactly what I was after.
