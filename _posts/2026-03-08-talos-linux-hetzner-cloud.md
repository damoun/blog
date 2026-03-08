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
2. **Provision the cluster** with Terraform using that snapshot

### Step 1: Build the Snapshot with Packer

The [Talos release page](https://github.com/siderolabs/talos/releases) provides raw disk images. For Hetzner, you need the `metal-amd64` image. Packer uploads it as a snapshot:

```hcl
source "hcloud" "talos" {
  image       = "debian-12"
  location    = "nbg1"
  server_type = "cx22"
  snapshot_name = "talos-v1.7.0"
}

build {
  sources = ["source.hcloud.talos"]

  provisioner "shell" {
    inline = [
      "wget -O /tmp/talos.raw.xz https://github.com/siderolabs/talos/releases/download/v1.7.0/hcloud-amd64.raw.xz",
      "xz -d /tmp/talos.raw.xz",
      "dd if=/tmp/talos.raw of=/dev/sda bs=4M",
      "sync",
    ]
  }
}
```

This boots a Debian server, writes the Talos raw image to disk, and Hetzner captures the result as a snapshot. You only need to do this once per Talos version.

### Step 2: Provision with Terraform

Once you have the snapshot ID, Terraform provisions the nodes and generates machine configs. The key resource is the machine config — one for control plane nodes, one for workers:

```hcl
resource "talos_machine_secrets" "this" {}

data "talos_machine_configuration" "controlplane" {
  cluster_name     = var.cluster_name
  cluster_endpoint = "https://${hcloud_server.controlplane[0].ipv4_address}:6443"
  machine_type     = "controlplane"
  machine_secrets  = talos_machine_secrets.this.machine_secrets
}

resource "talos_machine_configuration_apply" "controlplane" {
  for_each             = hcloud_server.controlplane
  client_configuration = talos_machine_secrets.this.client_configuration
  machine_configuration_input = data.talos_machine_configuration.controlplane.machine_configuration
  node = each.value.ipv4_address
}
```

The `talos` Terraform provider handles bootstrapping the cluster, generating the kubeconfig, and waiting for the nodes to be ready. You end up with a fully configured cluster without ever SSHing into anything.

## Module Structure

I keep this as a reusable Terraform module:

```
modules/talos-hcloud/
  main.tf          # servers, firewall rules
  talos.tf         # machine configs, bootstrap
  variables.tf
  outputs.tf       # kubeconfig, talosconfig
```

The module outputs the kubeconfig and talosconfig so they can be fed into subsequent Terraform or used directly with `kubectl` / `talosctl`.

## What I Learned

The biggest friction point is the Talos image version vs cluster version. You can't bootstrap a Talos v1.7 image with a mismatched installer. Keep a variable for the Talos version and reference it everywhere.

Also: Talos nodes don't have a way to install arbitrary kernel modules by default. If you need something like WireGuard or specific storage drivers, you need a custom Talos image built with [Image Factory](https://factory.talos.dev/). For my setup, I need a few extensions, so I build a custom image URL and reference it in both Packer and the machine config.

Once you get past the initial learning curve, operating Talos feels remarkably calm. Upgrades are `talosctl upgrade`, and you can watch the node drain, reboot, and rejoin without touching anything manually. For a cluster I don't want to babysit, that matters.
