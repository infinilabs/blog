---
title: "INFINI Console v1.28 Released"
meta_title: "INFINI Console v1.28 Released"
description: "Discover the new TopN feature and other enhancements in INFINI Console v1.28."
date: "2025-01-10T09:00:00Z"
categories: [ "Release", "News"]
author: "Medcl"
tags: ["Console", "TopN", "Release"]
draft: false
---

# INFINI Console v1.28 Released


We’re excited to announce **INFINI Console v1.28**, the latest update from **INFINI Labs**! This release brings the powerful **TopN** feature to help you identify key metrics efficiently, alongside other performance improvements and bug fixes. Read on for all the details and enhancements in this release.


## What is INFINI Console?

Great question! **INFINI Console** is a **lightweight, cross-version, unified management platform** designed specifically for search infrastructures. It empowers enterprises to:

- Manage multiple **search clusters** across different versions seamlessly.
- Gain centralized control for efficient cluster monitoring and maintenance.

With INFINI Console, you can streamline the management of your search ecosystem like never before! 🚀

Learn more here: https://docs.infinilabs.com/console/

---

## **Spotlight: Introducing the TopN Feature**

### **What is TopN?**
TopN, introduced in **Console v1.28.0**, allows you to quickly identify the top-ranking data points for critical metrics across multiple dimensions. Designed for performance optimization and decision-making, TopN offers faster, more precise monitoring even as cluster sizes grow.

In the past, monitoring solutions often struggled to handle large clusters with numerous nodes and indices, making it hard to pinpoint issues like the busiest nodes or the most delayed queries. With TopN, Console delivers an intuitive, efficient way to analyze your clusters, empowering users with:

- Multi-dimensional insights.
- Visualizations like **treemaps** for quick anomaly detection.
- Easy dynamic metric switching for deeper analysis.

---


### **Key Features**

#### **1. Multi-Dimensional Metrics Support**
Monitor critical metrics across indices, nodes, and shards:
- **Index-Level Metrics:** Indexing rate, query rate, indexing latency, query latency, etc.
- **Node-Level Metrics:** Indexing rate, query rate, CPU usage, JVM usage, etc.
- **Shard-Level Metrics:** Indexing rate, query rate, indexing latency, query latency, etc.

#### **2. Intuitive Visualizations**
- **Table View:** Displays detailed TopN data points.
- **Treemap View:** Visualizes data distribution with color and size dimensions for quick anomaly detection.
- **Dynamic Metric Switching:** Easily toggle between dimensions like size and color for customized insights.

#### **3. Customization and Extensibility**
- **Built-in Templates:** Preconfigured metric templates for common scenarios.
- **User-Defined Rules:** Create custom metric calculation rules for personalized analysis.

---

### **Use Cases**
- **Performance Bottleneck Analysis:** Identify hotspots during indexing or querying for faster troubleshooting.
- **Resource Optimization:** Detect resource-intensive nodes or shards to improve cluster efficiency.
- **Data-Driven Decisions:** Gain actionable insights for better operational and business outcomes.

For a detailed guide on using TopN, check out our [dedicated blog post](#).

---

## **What’s New in Console v1.28.0**

### **New Features**
- Added **TopN Metrics Querying** via the Insight API for fast identification of key data points.
- Log **cluster allocation explanations** into activity logs when cluster health turns red.
- Introduced new **segment memory metrics**, including `norms`, `points`, `version map`, and `fixed bit set`.
- Added **CRUD APIs** for managing custom Insight metrics.
- Preloaded several built-in metric templates for common use cases.

### **Bug Fixes**
- Resolved an issue with querying thread pool metrics when the cluster UUID was empty.
- Fixed unit test inconsistencies.

### **Optimizations**
- Addressed GitHub Issues **#46** and **#43** to improve CI pipelines.
- Enhanced **Agent List UI** for better data overflow handling.
- Added loading animations for each row in the overview table.
- Improved metric query bucket size configuration (#59).
- Added cluster version checks for `metric transport_outbound_connections` support.
- Set default timeout for the **DatePicker** component to 10 seconds.
- Enhanced **http_client** with more configuration options.

---

## **Get Started Today**

To quickly install and start using the INFINI Console, run the following command:

```
curl -sSL http://get.infini.cloud | bash -s -- -p console
```

Upgrade to **Console v1.28** and experience the enhanced monitoring and analytics capabilities firsthand. For more details, check out the full [release notes](https://docs.infinilabs.com/console/main/docs/release-notes/).

---

### **Connect with Us**
Have questions or need support? Reach out to our [technical support team](https://discord.gg/4tKTMkkvVX) or explore more about our products on the [INFINI Labs website](https://infinilabs.com).

Stay tuned for more updates and innovations from INFINI Labs!
