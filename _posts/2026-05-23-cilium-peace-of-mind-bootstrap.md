---
layout: single
title: "Getting Cilium Right from Day One: Recommended Bootstrap Configurations"
categories: [ops]
tags: [kubernetes, cilium, clustermesh, networking]
---

I've spent a lot of time recently bootstrapping Kubernetes clusters using Cilium as a Bring Your Own (BYO) CNI. I love it, it's fast, feature-rich, and Hubble's visibility feels like magic compared to legacy CNIs.

But infrastructure setups are rarely plug-and-play. While the Cilium CLI is fine for quick tests, it tends to hide too much under the hood by wrapping kubectl and helm. For production, managing it declaratively through Helm and GitOps is the way to go. There are a few configuration details that are worth getting right from day one. If you skip them during bootstrap thinking you'll sort it out later, you're looking at some annoying migration work down the road.

These are the baseline configurations I now set up on every new cluster.

## 1. Cluster IDs and Names (ClusterMesh Prep)

When you set up a fresh cluster, it's easy to ignore `cluster.id` and `cluster.name`. If you only have one cluster, defaults are fine.

But if you eventually connect clusters using ClusterMesh, you'll run into a wall: each cluster needs a unique ID (between 1 and 255) and name. 

Changing them later isn't impossible, but it is a chore. You have to rotate the `cilium-ca` certs and restart the Cilium agents and operators across your nodes. It's much easier to just define them upfront in your Helm values:

```yaml
cluster:
  id: 1
  name: my-prd-cluster
```
{: .notice--success}

Standardizing your CA across your fleet from the start saves a lot of pain. Instead of letting each cluster generate its own CA, use a pre-shared `cilium-ca` secret everywhere. This makes establishing trust in ClusterMesh automatic later on.
{: .notice--info}

## 2. IPAM and the Pod CIDR Sweet Spot

IPAM works invisibly until you run out of IPs, which is usually a bad day. Cilium supports several IPAM modes, but for my setups, the IPAM operator with `ClusterPool` is the easiest to reason about.

By default, Cilium allocates a massive `10.0.0.0/8` block. This is a classic trap: it will likely overlap with your cloud VPC or local networks, causing weird routing issues. And if you mesh multiple clusters, you'll get immediate collisions. Define a dedicated, non-overlapping `podCIDR`. It's overlay traffic, so it doesn't consume your routable VPC IPs anyway, you can be generous.

A `/18` block is a good starting point for a cluster. Splitting it with a node mask size (`clusterPoolIPv4MaskSize`) of `/25` gives a nice balance.

```yaml
ipam:
  operator:
    clusterPoolIPv4PodCIDRList:
      - 10.0.0.0/18
    clusterPoolIPv4MaskSize: 25
```

Why `/25`? The math is simple: a `/18` network provides 16,384 IPs. A `/25` allocates 128 IPs per node, supporting up to 128 nodes. Since the Kubernetes large cluster guide recommends a maximum of 110 pods per node anyway, `/25` fits perfectly with a bit of headroom. 

This gives you enough space to run a ~60-node cluster with plenty of room for blue-green cluster upgrades or scaling. You can go larger if you expect thousands of nodes, but a `/18` is a solid default for most setups.

Keep in mind that while you can add more CIDR blocks later, the node mask size (`clusterPoolIPv4MaskSize`) is immutable. Changing it later means draining and recreating all your nodes to allocate new subnets, so get this right first.
{: .notice--info}

## 3. Automating Configuration Rollouts

If you manage deployments via GitOps, you want config changes to apply automatically. By default, updating the `cilium-config` ConfigMap updates the config, but the running Cilium pods keep using the old settings until you manually restart them.

You can automate this by enabling the rollout flags in the Helm chart. This adds a ConfigMap checksum annotation to the pod templates, forcing Kubernetes to perform a rolling update of the DaemonSet and operator deployments whenever the configuration changes.

```yaml
rollOutCiliumPods: true
operator:
  rollOutPods: true
envoy:
  rollOutPods: true
```
{: .notice--success}

## 4. Hubble TLS Certificate Renewal

Hubble is great for network visibility, but it relies on TLS certificates. By default, the Helm chart generates these certs once during install.

These default certificates expire in 3 years (1095 days) and don't auto-rotate. Three years down the road, your observability stack will silently break when the certs expire.

Switch the generation method to `cronJob`. This deploys a helper CronJob that automatically checks and renews the certificates before they expire.

```yaml
hubble:
  tls:
    auto:
      method: cronJob
```
{: .notice--warning}

As a bonus, the certificates bake in the `cluster.name` in their SAN. If you ever rename the cluster later, the CronJob will automatically update the certificates with the correct names.

## 5. Protecting Cilium Pods from Eviction

During heavy resource pressure, the kubelet will evict pods to keep the node alive. 

Because Cilium runs its datapath in the kernel via eBPF, losing the `cilium-agent` doesn't drop existing traffic. Packets still flow. However, the node's network control plane freezes: new pods won't get interfaces programmed, Service routing maps become stale, and Hubble goes dark. 

By default, the Helm chart leaves `priorityClassName` empty. To protect the agent from eviction, assign it a system priority class:

```yaml
priorityClassName: system-node-critical
```
{: .notice--success}

This ensures the scheduler keeps the Cilium agent running even when the node is heavily overloaded.

## Wrapping Up

Getting these basic Helm values defined early saves a lot of operational work down the line. It's tempting to rush through CNI setup, but spending 10 minutes planning your IPAM layout, hardcoding cluster names, and enabling auto-rotation for certs will keep your cluster stable as it scales.

If you're bootstrapping new clusters, configure these from day one so you don't have to cycle nodes or rotate certs under pressure later.

[Check out the Cilium Docs](https://docs.cilium.io/en/stable/){: .btn .btn--primary .btn--large}
