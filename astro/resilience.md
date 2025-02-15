---
sidebar_label: 'Resilience'
title: "Resilience"
id: resilience
description: Learn how Astronomer leverages Availability Zones to make the control plane and data plane resilient.
---

The Astro control and data planes are architected and deployed on major public clouds to take advantage of their resilient and highly available regions. Regions provide multiple physically separated and isolated Availability Zones (AZs) which are connected through low-latency, high-throughput, and highly redundant networking within a geographic region. Additionally, each AZ has independent power, cooling, and physical security.

Astro leverages AZs for both the control and data planes. The control plane leverages 3 AZs per region, while the data plane leverages 2 AZs per region on AWS, and 3 AZs per region on GCP. control planes and data planes are also segregated by cloud providers, with plans to support more public clouds for the data plane. As a result, both planes are expected to survive and recover from an AZ outage, though they may experience some degradation until the impacted resources are re-provisioned/promoted to a non-impacted AZ.

In the case of a full region failure of the control plane, the services and data would be recovered in an alternate region by Astronomer. In the case of a full region failure of the data plane, clusters impacted by a full regional outage will be restored to an alternate region (see [Disaster recovery](disaster-recovery.md) for details).

Astro also provides DDoS protection with always-on traffic monitoring, detection, and automatic attack mitigations for all inbound network traffic, along with suspicious IP throttling and brute-force protection as part of our secure authentication service.
