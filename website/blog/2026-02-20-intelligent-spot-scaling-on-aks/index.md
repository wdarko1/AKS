---
title: "Intelligent Spot Scaling on AKS with Node Auto Provisioning or Cluster Autoscaler"
description: "Learn how to optimize Spot node scaling with the Spot Placement Score API paired with either Node Auto Provisioning or Cluster Autoscaler. Also learn best practices for spot node scaling in AKS."
date: 2026-02-20
authors: ["wilson-darko"]
tags:
  - node-auto-provisioning
  - cluster-autoscaler
---

## Using Spot in Azure Kubernetes Services
---

<!-- truncate -->

:::info

Try out the OSS Intelligent Spot Scaling on AKS project [](https://github.com/iamvighnesh/intelligent-spot-scaling-on-aks) by [@iamvighnesh](https://github.com/iamvighnesh/)

:::

---

Overview
Running workloads on Azure spot Virtual Machines (VMs) offers massive cost savings—often exceeding 80–90% compared to on‑demand compute. But Spot capacity is unpredictable. Evictions can occur at any moment, and choosing the wrong VM SKU in a region can drastically increase disruption rates.
Enter Intelligent Spot Scaling: an open‑source solution integrating the Azure Spot Placement Score API with Cluster Autoscaler (CA) and Node Auto Provisioning (NAP) to dynamically pick the best Spot VM SKUs—right when your cluster needs them.
👉 GitHub project: https://github.com/iamvighnesh/intelligent-spot-scaling-on-aks
This project was created to address challenges reported by large enterprise customers—including financial institutions such as Morgan Stanley and UBS—who run highly burstable, compute‑intensive workloads on Spot and face issues with unpredictable availability.
This blog details the design, architecture, and customer value of intelligent Spot selection on AKS.

Why Spot Workloads Are Hard
Spot VMs provide huge savings, but come with challenges:

Eviction risk (as low as 30 seconds notice today)
Regional SKU availability volatility
Entire Spot pools sometimes unavailable
Manual SKU selection quickly becomes outdated

Even when Spot capacity does exist, choosing the wrong SKU leads to:

Higher eviction rates
Failure to scale
Cascading workload delays

This is why a data‑driven approach is needed.

### The Azure Spot Placement Score API

The Spot Placement Score API rates VM sizes in a region as:

- High
- Medium
- Low

These scores reflect relative likelihood of successful deployment and sustained availability based on recent platform trends.
The intelligent‑scaling project consumes these scores and ranks candidate SKUs accordingly. This allows AKS to:

Prefer SKUs with a High score
Avoid capacity traps
Pivot to fallback SKUs in real time

This is the core innovation powering intelligent Spot scaling.

Solution Architecture
The repository supports two approaches depending on your AKS scaling strategy.
1. Cluster Autoscaler (Priority Expander) Approach
For existing AKS clusters with pre‑created node pools.
How it works:

A GitHub Actions workflow runs a script (scripts/spot-score.sh).
The workflow calls the Spot Placement Score API.
Scores are tiered into High / Medium / Low.
The script updates:

PriorityClass objects (spot‑preferred → spot‑fallback)
cluster-autoscaler-priority-expander ConfigMap


Cluster Autoscaler then:

Scales High‑score Spot node pools first
Falls back to Medium/Low
Ultimately uses on‑demand pools when Spot is unavailable



This improves reliability by avoiding node pools that look “cheap” but are actually risky.

2. Node Auto Provisioning (NAP) Approach
For new or modern AKS clusters using dynamic autoscaling (Karpenter-based NAP).
How it works:

Pending pods trigger NAP to provision compute dynamically.
A control script generates NodePool manifests using Spot scores.
NAP selects the optimal Spot VM SKU at provisioning time.
If Spot becomes risky, NAP:

Switches families
Falls back to on‑demand
Rebalances across zones



This makes Spot usage hands‑free: no predefined node pools, just inte
