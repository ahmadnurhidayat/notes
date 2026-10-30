---
tags:
  - aws
  - route53
  - dns
  - networking
  - vpc
---

# Route 53 VPC Resolver — Best Practices

Operational best practices for Amazon Route 53 **VPC Resolver** — the DNS
resolution layer inside a VPC, including inbound and outbound Resolver
endpoints. The goal is reliable, scalable, and secure DNS resolution for
workloads and resources.

!!! info "Source"
    Condensed from the AWS docs:
    [Best practices for VPC Resolver](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/best-practices-resolver.html).
    Content was rephrased for compliance with licensing restrictions. Always
    check the AWS documentation for the authoritative, current guidance.

## 1. Avoid loop configurations with Resolver endpoints

A routing loop occurs when the **same VPC** is associated with both a Resolver
rule *and* that rule's inbound endpoint — queries can circle back on
themselves.

- Do not associate a VPC with both a Resolver rule and its own inbound endpoint.
- Use **AWS RAM** to share VPCs across accounts while keeping the routing
  configuration correct and loop-free.

!!! warning
    Loops are easy to create in multi-account setups where rules and endpoints
    are shared. Map the query path end to end before associating rules.

## 2. Scale Resolver endpoints

Query throughput is bounded by connection tracking and by unique
source IP/port combinations.

- Write **security group rules that allow traffic based on connection state**
  so the connection-tracking overhead stays low.
- Follow the recommended inbound and outbound security group rules for Resolver
  endpoints to maximise query throughput.
- **Monitor the unique IP address and port combinations** generating DNS
  traffic so you can spot approaching capacity limits before they bite.

## 3. High availability for Resolver endpoints

- Create inbound endpoints with IP addresses in **at least two Availability
  Zones** for redundancy.
- **Provision additional network interfaces** so the endpoint keeps serving
  during maintenance windows or traffic surges.

!!! tip
    Treat endpoint ENIs like any other capacity: size for peak plus headroom,
    and spread across AZs so a single-AZ event does not take DNS down.

## 4. Prevent DNS zone walking attacks

**Zone walking** is an attack where someone attempts to enumerate all records in
a DNSSEC-signed zone.

- Be aware that DNSSEC-signed zones can be targeted for enumeration.
- If your endpoints get **throttled** because of suspected zone walking,
  contact **AWS Support** for help.

## 5. Subnet compatibility for Resolver endpoints

- Prefer **non-Outposts subnets** for Resolver endpoints.
- **Outposts subnets with Local Network Interface (LNI) enabled are not
  compatible** with VPC Resolver endpoints.
- Enabling LNI on a subnet that already contains Resolver endpoint ENIs will
  cause **those ENIs to stop functioning**.

!!! danger
    Do not enable LNI on a subnet hosting Resolver endpoint ENIs — it breaks
    DNS resolution for everything depending on that endpoint.

## Summary

| Area | Do this |
| :--- | :--- |
| **Loops** | Never pair a VPC with both a Resolver rule and its own inbound endpoint; share VPCs via AWS RAM. |
| **Scaling** | State-based security group rules; follow recommended rules; monitor unique IP/port pairs. |
| **HA** | Inbound endpoints across ≥2 AZs; add network interfaces for surges/maintenance. |
| **Security** | Watch for zone walking on DNSSEC zones; contact AWS Support on throttling. |
| **Subnets** | Use non-Outposts subnets; never enable LNI on subnets with Resolver ENIs. |

---

*See also:* [Split-Horizon DNS Forwarding](../system/split-horizon-dns-forwarding.md)
· [Unbound DNS Setup](../system/unbound-dns-setup.md)
