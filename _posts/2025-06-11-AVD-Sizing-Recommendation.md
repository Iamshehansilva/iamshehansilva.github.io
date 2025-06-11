---
title: "Balancing Performance and Cost Optimization in AVD"
date: "2025-06-11 00:00:00 +0100"
categories: [Azure, AVD, SKU, Compute, Cost, Optimization,Nerdio,Auto-Scale]
toc: true
tags: [Azure, AVD, SKU, Compute, Cost, Optimization,Nerdio,Auto-Scale]
image:
  path: /assets/img/AVD-Sizing/AVD-Cost.png
  src: /assets/img/AVD-Sizing/AVD-Cost.png
---

## 🕵️‍♂️ Situation

One of the most common queries I receive from customers is how to optimize costs in Azure Virtual Desktop (AVD) while maintaining performance. Microsoft's Azure Advisor often recommends B-Series VMs for underutilized workloads, including AVD. **But should you consider switching to low-cost SKUs like B-Series for AVD session hosts? 💡 My Take: The Short Answer is NO! Here's Why**

## 📌 Why not B-Series Family

While B-Series virtual machines may seem attractive from a cost perspective, they come with significant performance limitations that make them unsuitable for AVD deployments. Let’s break down the key concerns.

1️⃣ **B-Series VMs use burstable CPU, meaning they accumulate CPU credits during low usage periods and use them later for performance bursts.**

**The problem?** AVD requires consistent and predictable performance, especially during peak user activity. Once CPU credits are exhausted, performance drops drastically, leading to:
   - Slow session responsiveness
   - Poor user experience
   - Increased support tickets from frustrated users

2️⃣ **Risk of CPU Throttling**

When a B-Series VM runs out of CPU credits, it gets throttled to baseline performance , which can be as low as 10-15% of the allocated CPU capacity. This can result in:
   - Laggy user sessions
   - Freezing or unresponsive applications
   - Session disconnections due to resource exhaustion

3️⃣ **Not Scalable for Multi-User Environments**

B-Series VMs are designed for workloads with low CPU usage, such as small web servers or development/test environments. They are not built to handle multiple concurrent AVD sessions, which require consistent CPU and RAM availability.

## 🎯 What Are the Right VM Series Family for AVD?

Imagine you're setting up Azure Virtual Desktop (AVD) for your company. You want it to run smoothly, ensuring a great experience for users, but at the same time, you don’t want to overspend on resources.

Microsoft provides [AVD sizing guidelines](https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/virtual-machine-recs) , which are a great starting point.

From my experience, I usually go with D-Series, F-Series, or N-Series VMs because they offer a solid balance of performance and cost. But sometimes, I’ve run into situations where RAM, not CPU, was the bottleneck. In those cases, switching to an E-Series VM made more sense instead of moving up to a larger D-Series VM with more CPU than needed and a higher price tag. Always do cost comparison with differents vm faimiles.

🚀 One key takeaway: Always test, test, test! With great power comes great responsibility!

## 📌 Cost Optimization Strategies for AVD Without Compromising Performance

While choosing the right VM series family is crucial, there are several other ways to reduce costs while maintaining a great user experience. Let’s take an example environment and explore the best cost-saving strategies:

Example AVD Environment
- Host Pool Size: 10 VMs
- Always-on VMs: 2 VMs running 24/7 (full-time)
- Dynamic VMs: 8 VMs running 440 hours per week
- Session Density: 8 users per VM
- Peak Hours: 07:00 – 18:00
- Off-Peak Hours: 18:00 – 07:00

### 1️⃣ Reserved Instances (RIs)
1-Year or 3-Year Reserved Instances (RIs)
   - Since 2 VMs run 24/7 (720 hours per month each), purchasing 1-year or 3-year RIs for these VMs can maximize cost savings.

### 2️⃣ Review & Optimize Autoscaling Configuration

Scaling configuration should align with actual usage patterns. Here’s how:
- Analyze active vs. disconnected sessions during off-peak hours.
   - Example:
     - Off-peak hours start at 18:00, but data shows that at 16:00, only 10 sessions are active while 10 VMs are still running.
     - Instead of scaling in at 18:00, consider scaling in at 16:00 or 17:00 for additional savings.

### 3️⃣ Review Host Pool Usage
1. If the total number of AVD users has decreased, consider downsizing the base capacity of your host pool.
2. While demand can fluctuate, it's unlikely that a significant increase/decrease in users will occur overnight, so regularly reassessing the base capacity can help optimize costs.

### 4️⃣ Optimizing Session Limits for Efficient Scale-In

- One of the easiest ways to reduce unnecessary costs in AVD is by optimizing session limit policies to ensure that idle or inactive sessions don’t keep consuming resources. Properly configuring these policies allows faster scale-in, freeing up VMs when they are no longer needed.
  - Log off disconnected sessions after X minutes/hours
  - Disconnect idle sessions after X minutes/hours
  - Log off empty RemoteApp sessions after X minutes/hours

## 📈 How to Analyze Your AVD Environment?

### 🔹Using Nerdio for Auto-Scale Insights
 - Nerdio Auto-Scale History provides a detailed overview of when VMs scale in/out, helping you adjust auto-scale configuration accordingly.
[Nerdio Auto-scale History](https://nmehelp.getnerdio.com/hc/en-us/articles/26124306148237-Auto-scale-History-for-Dynamic-Host-Pools)

### 🔹 Using Log Analytics & KQL Queries

 - **Active Session Hosts by Heartbeat**
 
```KQL
Heartbeat
| summarize RunningHosts=dcount(Computer) by bin(TimeGenerated, 1h)
| order by TimeGenerated asc

```

- **Active Session Hosts by Heartbeat (Filtered by Resource Group)**

```KQL
let HostPoolResourceGroup = " Resource Group ";  // Replace with actual resource group name
let TotalHosts = toscalar(
    Heartbeat
    | where ResourceGroup == HostPoolResourceGroup
    | summarize Total=dcount(Computer)
);
Heartbeat
| where ResourceGroup == HostPoolResourceGroup
| summarize RunningHosts=dcount(Computer) by bin(TimeGenerated, 1h)
| extend NotActiveHosts = TotalHosts - RunningHosts
| order by TimeGenerated asc

```
![Active Session Hosts](/assets/img/AVD-Sizing/AVD-Sizing-01.png)
- **Active vs. Disconnected User Sessions**

```KQL
WVDConnections
| summarize ActiveUsers = countif(State == "Connected"), 
            DisconnectedUsers = countif(State == "Completed") 
            by bin(TimeGenerated, 30m)
| order by TimeGenerated desc

```
![Active Session Hosts](/assets/img/AVD-Sizing/AVD-Sizing-02.png)

⭐ [Session Lifecycle More info](https://learn.microsoft.com/en-us/azure/virtual-desktop/diagnostics-log-analytics#cadence-for-sending-diagnostic-events)

## 🌟 Summary

One of the most common questions we get from customers is how to optimize their AVD costs. B-Series VMs? Sure, they can reduce costs, but what about performance? The short answer is no, they’re not suitable due to their burstable CPU design, inconsistent performance, and limitations in multi-user environments.

Based on my experience, when it comes to cost optimization, it’s important to look beyond just cheap VM SKUs. Focus on strategies like reserved Instances, autoscaling configuration reviews, session limit policies, and regular host pool usage analysis to ensure cost-efficiency without compromising user experience.

Key takeaway: Always prioritize user experience first—then decide which cost-saving strategies best fit your environment.

Try out these options and see how they work for your specific scenario. You may have different insights or encounter factors I haven’t considered. Keep exploring....

