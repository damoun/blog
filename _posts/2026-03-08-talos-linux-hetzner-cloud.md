---
layout: single
title: Running Talos Linux on Hetzner Cloud
categories: [ops]
tags: [talos, hetzner, terraform, kubernetes]
---

I've run Kubernetes on a lot of things. Plain VMs with kubeadm. k3s on Raspberry Pis. Managed clusters on every major cloud. But at some point you start wanting more control over the OS itself — and that's where [Talos Linux](https://www.talos.dev/) comes in.

Talos is not a normal Linux distribution. There is no SSH. No shell. No package manager. The entire OS is read-only and managed through a gRPC API (`talosctl`). It exists for one purpose: running Kubernetes. That constraint, which sounds limiting, is actually what makes it interesting.

## Why Talos?

Most Kubernetes distributions assume a general-purpose OS underneath. That means you're responsible for patching the OS, hardening it, managing SSH keys, and keeping drift from accumulating over time. Talos eliminates that surface entirely. The attack surface is minimal, upgrades are atomic and rollback-able, and every node is configured identically from a machine config file.

For my homelab/VPS setup on Hetzner Cloud, this was appealing for a specific reason: Hetzner doesn't offer a managed Kubernetes service that I actually want to use. So I'm provisioning my own nodes, and I want them to be as boring and uniform as possible.

## The Two-Step Approach

Hetzner doesn't have Talos images in its catalog. So the workflow is:

1. **Build a Talos snapshot** with Packer
2. **Provision the infrastructure** with Terraform using that snapshot

The source code for all of this lives in [damoun/terraform-hcloud-talos](https://github.com/damoun/terraform-hcloud-talos).

### Step 1: Build the Snapshot with Packer

Talos provides pre-built images through [Image Factory](https://factory.talos.dev/), which lets you include custom extensions (WireGuard, storage drivers, etc.) in the image. Packer boots a temporary Debian server in rescue mode, writes the Talos image directly to disk, and Hetzner captures the result as a labeled snapshot:

```hcl
variable "talos_version" {
  default = "v1.12.4"
}

local {
  image_url = "https://factory.talos.dev/image/ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515/${var.talos_version}/hcloud-amd64.raw.xz"
}

source "hcloud" "talos" {
  location      = "fsn1"
  server_type   = "cx22"
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

The key detail here is rescue mode. Instead of booting the Debian image normally, Packer boots into Hetzner's rescue system, which gives direct access to `/dev/sda`. The Talos image gets written straight to disk. You only need to do this once per Talos version.

The long hash in the Image Factory URL is a schema ID that defines which extensions to include. You can generate your own at [factory.talos.dev](https://factory.talos.dev/).

### Step 2: Provision with Terraform

Terraform looks up the Packer snapshot by its labels and provisions the cluster infrastructure — network, firewall, placement groups, and servers:

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

Notice there's no Talos provider in this module. The Terraform module only provisions the Hetzner infrastructure — servers, networking, firewall rules, and an optional load balancer for the Kubernetes API on port 6443. Talos bootstrapping (generating machine configs, applying them, bootstrapping etcd) happens separately with `talosctl` or with the Talos Terraform provider in a consuming module.

This separation keeps the infrastructure module focused and reusable. The module outputs control plane and worker IPs (both public and private), which you feed into whatever handles the Talos configuration step.

## Module Structure

The module is structured as a standard Terraform module:

```
terraform-hcloud-talos/
  main.tf            # servers, network, firewall, placement groups
  load_balancer.tf   # optional LB for Kubernetes API (port 6443)
  variables.tf       # cluster_name, talos_version, counts, CIDRs
  outputs.tf         # control plane/worker IPs (public, private, IPv6)
  terraform.tf       # provider requirements
  packer/
    talos.pkr.hcl    # snapshot build definition
```

Placement groups with spread policy ensure control plane nodes and workers land on different physical hosts. The private network (`172.16.16.0/20` by default) keeps inter-node traffic off the public internet.

## What I Learned

The biggest friction point is the Talos image version. The Packer snapshot version and the Talos installer version used during bootstrap must match. I keep a single `talos_version` variable referenced in both Packer and Terraform to avoid drift.

Talos nodes don't support arbitrary kernel modules out of the box. If you need something like WireGuard or specific storage drivers, you need a custom image from [Image Factory](https://factory.talos.dev/). That's why the Packer template uses a factory URL with a specific schema ID rather than the vanilla release image.

The firewall is deliberately minimal — only port 50000 (Talos apid) from admin CIDRs and ICMP. Kubernetes API access (port 6443) goes through the optional load balancer or direct node access, depending on your setup.

Once you get past the initial learning curve, operating Talos feels remarkably calm. Upgrades are `talosctl upgrade`, and you can watch the node drain, reboot, and rejoin without touching anything manually. For a cluster I don't want to babysit, that matters.
