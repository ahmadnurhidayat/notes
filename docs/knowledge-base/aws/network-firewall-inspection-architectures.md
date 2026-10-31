---
tags:
  - aws
  - networking
  - network-firewall
  - security
  - transit-gateway
  - vpc
---

# Choosing an AWS Network Firewall Inspection Architecture

AWS Network Firewall now offers **three** deployment patterns for multi-VPC,
multi-account inspection. This guide compares them — traffic flow, cost, and a
decision framework — so you can pick without trial and error.

!!! info "Source"
    Condensed from the AWS blog
    [Choosing the right inspection architecture for AWS Network Firewall](https://aws.amazon.com/blogs/networking-and-content-delivery/choosing-the-right-inspection-architecture-for-aws-network-firewall/).
    Content was rephrased for compliance with licensing restrictions. Check AWS
    docs and pricing for authoritative, current details.

## The three patterns

| Pattern | Launched | Scope | One-liner |
| :--- | :--- | :--- | :--- |
| **1. Traditional Inspection VPC** | 2020 | Centralized | Dedicated VPC of firewall endpoints; all traffic routes through it via Transit Gateway / Cloud WAN. |
| **2. Multiple VPC Endpoints** | May 2025 | Distributed | Secondary firewall endpoints pushed into spoke VPCs, sharing one central policy. |
| **3. Transit Gateway Native Attachment** | GA Jun/Jul 2025 | Centralized | Firewall attaches *directly* to Transit Gateway as a managed network-function attachment — no inspection VPC. |

## Quick comparison

| Dimension | Inspection VPC | Multiple VPC Endpoints | TGW Native Attachment |
| :--- | :--- | :--- | :--- |
| **East-West inspection** | Yes (TGW / Cloud WAN) | Not applicable | Yes (TGW; **no** Cloud WAN) |
| **North-South inspection** | Full | Full | Full |
| **TLS inspection** | Supported | **Not supported** | Supported |
| **Source IP visibility** | Preserved via TGW / Cloud WAN | Preserved (local) | Preserved via TGW |
| **Multi-account** | TGW/Cloud WAN + RAM | Firewall sharing (RAM) | TGW sharing + RAM |
| **Data-plane scope** | Large (centralized) | Small (distributed) | Large (centralized) |

## Pattern 1 — Traditional Inspection VPC

A dedicated VPC holds the Network Firewall endpoints. All east-west and
north-south traffic routes through Transit Gateway (or Cloud WAN core) into that
VPC, is inspected, then forwarded on.

**East-West flow (simplified):** spoke VPC-A → TGW (pre-inspection RT) →
Inspection VPC firewall endpoint → back to TGW (post-inspection RT) → spoke
VPC-B. Enable **appliance mode** on the Inspection VPC attachment to keep flow
symmetry.

**North-South flow:** spoke → TGW → firewall endpoint → NAT Gateway → Internet
Gateway → internet; return traffic reverses, with NAT stateful tracking and
firewall stateful match.

!!! tip "When to use"
    You need granular routing control in a dedicated VPC (e.g. different
    policies per direction, or separate N-S vs E-W paths), **or** you already
    have Transit Gateway infrastructure with established runbooks and team
    expertise.

Trade-off: you manage extra VPCs, subnets, route tables, and NAT Gateways.

## Pattern 2 — Multiple VPC Endpoints

Instead of funneling everything through a central VPC, you deploy **secondary
firewall endpoints** directly into spoke VPCs via VPC endpoint associations. One
firewall, one **shared policy**, distributed data paths.

!!! note "Not the same as distributed firewalls"
    A distributed model gives each VPC its *own* firewall and policy. Here a
    **single firewall** extends secondary endpoints into spokes — cheaper
    (secondary endpoints are priced lower) and **no policy drift**. You cannot
    apply different rules to different endpoints of the same firewall.

**Cross-account setup:**

1. Account A creates the Network Firewall in VPC-A and assigns the policy.
2. Account A shares the firewall with Account B via **AWS RAM**.
3. Account B creates a VPC endpoint association in VPC-B — a secondary endpoint
   in the local FW subnet, enforcing Account A's policy. Secondary endpoints can
   only exist in AZs where a **primary endpoint already exists**.

**North-South flow:** app subnet (`0.0.0.0/0 → vpce-id`) → local firewall
endpoint → NAT Gateway → Internet Gateway; return reverses.

**East-West:** still needs Transit Gateway or Cloud WAN for inter-VPC
connectivity. Inspection happens at the *local* endpoint before traffic crosses
the inter-VPC path — but you're then maintaining per-VPC endpoints *and*
depending on TGW. For E-W, a centralized pattern is usually better.

!!! tip "When to use"
    Primary concern is **north-south** inspection; public-facing workloads (ALBs,
    public EC2) that route straight through a local endpoint to an Internet
    Gateway; high data-transfer volumes where avoiding TGW processing fees pays
    for the secondary endpoints; or you want **per-VPC blast-radius isolation**
    with one central policy.

!!! warning
    **TLS inspection is not supported** for firewalls with VPC endpoint
    associations.

## Pattern 3 — Transit Gateway Native Attachment

The simplest centralized option. The firewall attaches **directly to Transit
Gateway** as a network-function attachment. AWS provisions and manages the
underlying resources — **no separate Inspection VPC, firewall subnets, or
endpoint management**. You specify the AZs and create the policy.

- **Appliance mode is always on** → symmetric routing guaranteed, no manual
  config.
- **AWS manages multi-AZ redundancy** automatically.
- Cross-account via RAM: share the Transit Gateway; the firewall owner creates
  the attachment.

**East-West flow:** spoke VPC-A (`0.0.0.0/0 → TGW`) → TGW pre-inspection RT →
firewall native attachment → TGW post-inspection RT → VPC-B.

**North-South flow:** spoke → TGW pre-inspection RT → firewall attachment → TGW
post-inspection RT → **Egress VPC** attachment → NAT Gateway → Internet Gateway.

!!! tip "When to use"
    New deployments needing centralized E-W **and** N-S inspection, or existing
    TGW environments wanting to simplify. Same capabilities as Inspection VPC
    (TLS inspection, stateful rule groups, Suricata-compatible IPS) with far less
    operational overhead.

!!! warning "Important limits"
    - **Not supported with AWS Cloud WAN** (as of writing).
    - Centralized internet egress still needs a **separate Egress VPC** (NAT GW +
      IGW).
    - Centralized egress here incurs **double TGW data-processing charges** —
      traffic is processed spoke→firewall, then firewall→egress VPC.

## Cost comparison

| Cost component | Inspection VPC | Multiple VPC Endpoints | TGW Native Attachment |
| :--- | :--- | :--- | :--- |
| NFW **primary** endpoint hourly (per AZ) | Yes | Yes | Yes |
| NFW **secondary** endpoint hourly (per AZ, reduced) | — | Yes (per VPC) | — |
| NFW data processing (per GB) | Yes | Yes (**none** for secondary endpoint) | Yes |
| TGW attachment hourly | Yes | — | Yes |
| TGW data processing (per GB) | Yes | — | Yes |
| NAT Gateway hourly + data | Waived¹ | Waived¹ (per VPC) | Waived¹ |

¹ **NAT Gateway discount:** standard NAT GW per-hour and data-processing charges
are waived one-to-one when the NAT Gateway is in the **same Region and same
networking path** as the Network Firewall endpoint (primary or secondary). See
AWS Network Firewall pricing.

### At scale (many VPCs)

- **Patterns 1 & 3** have similar infra cost, driven by TGW per-attachment fees.
  **Pattern 3 costs more for N-S** because the firewall is a separate attachment
  and egress needs its own VPC, so N-S traffic crosses TGW **twice** (one extra
  attachment + one extra data-processing hop vs Pattern 1).
- Pattern 3's real win is **operational**: no inspection VPC to design, deploy,
  monitor, or maintain.
- **Pattern 2 is most expensive at scale** — each new VPC needs its own
  secondary endpoints (one per AZ), accumulating as VPC count grows.

### At small scale (≤4 VPCs)

- **Pattern 2 is the most cost-effective.** Dropping Transit Gateway entirely
  removes both attachment and data-processing fees, offsetting secondary
  endpoint cost. Great when you only need N-S inspection across a few VPCs.

## Decision framework

```
Need AWS Cloud WAN compatibility?
├─ Yes → Inspection VPC (Pattern 1)   # TGW Native Attachment not supported with Cloud WAN
└─ No
   ├─ Need TLS inspection?
   │  └─ Yes → Pattern 1 or 3 (NOT Pattern 2)
   ├─ Only north-south, and ≤4 VPCs (or very high data-transfer, public-facing)?
   │  └─ Yes → Multiple VPC Endpoints (Pattern 2)
   ├─ Need granular per-direction routing / separate N-S and E-W paths?
   │  └─ Yes → Inspection VPC (Pattern 1)
   └─ Otherwise (centralized E-W + N-S, least ops overhead)
      └─ Transit Gateway Native Attachment (Pattern 3)   # default for most
```

!!! tip "Default"
    For most customers, **Transit Gateway Native Attachment** is the simplest
    operational model — AWS runs the endpoints in a managed VPC, handles multi-AZ
    redundancy and symmetric routing, and removes the inspection-VPC toil.

## Considerations

- **Appliance mode** — manual on the Inspection VPC attachment (Pattern 1);
  **always on** for TGW Native Attachment (Pattern 3).
- **TLS inspection** — Patterns 1 & 3 only; **not** Pattern 2.
- **Cross-account** — all three via AWS RAM: share the TGW (Patterns 1 & 3) or
  the firewall policy (Pattern 2).
- **Cost allocation** — TGW supports flexible allocation at TGW, attachment, and
  flow levels, so you can distribute costs across accounts (Patterns 1 & 3).
- **Migration risk** — Inspection VPC is **not deprecated**; migrate only when
  operational simplification beats the effort. **Test with a non-production VPC
  first.**

---

*See also:* [Egress Inspection with GWLB](egress-inspection-gwlb.md)
· [Centralized Egress (TGW + NAT)](centralized-egress-tgw-nat.md)
