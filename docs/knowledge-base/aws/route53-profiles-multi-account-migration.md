---
tags:
  - aws
  - route53
  - dns
  - networking
  - vpc
  - multi-account
---

# Migrating Multi-Account DNS to Route 53 Profiles

How to move a multi-account, multi-VPC DNS setup to **Amazon Route 53
Profiles** — a single container that groups DNS configuration (Private Hosted
Zone associations, Resolver forwarding rules, DNS Firewall rules) and can be
shared across accounts via AWS RAM and associated to VPCs in one Region.

!!! info "Source"
    Condensed from the AWS blog
    [Migrating your multi-account DNS environment to Amazon Route 53 Profiles](https://aws.amazon.com/blogs/networking-and-content-delivery/migrating-your-multi-account-dns-environment-to-amazon-route-53-profiles/).
    Content was rephrased for compliance with licensing restrictions. Check the
    AWS documentation and blog for authoritative, current guidance.

## Why Route 53 Profiles

Before Profiles, associating Private Hosted Zones (PHZs) and Resolver rules to
VPCs across accounts meant custom API-driven automation and a fragmented sharing
experience, especially when onboarding new accounts and VPCs.

A **Profile** groups DNS configuration into one object you manage centrally,
share with AWS RAM, and attach to VPCs. Once a Profile is associated with a VPC,
it answers that VPC's DNS queries based on the Profile's settings.

!!! note "Prioritization"
    If a VPC has both a **direct** PHZ (or Resolver rule) association *and* a
    Profile that contains the same PHZ/rule, the **direct association wins**.
    For conflicting domain names, the **most specific** match wins.

## Starting environments

Two common multi-account layouts, both resolving within AWS and between AWS and
on-premises (connected via Direct Connect or Site-to-Site VPN):

| Approach | Where PHZs live | Resolver endpoints/rules |
| :--- | :--- | :--- |
| **PHZs in Networking Account** | Central networking account owns all PHZs, associated to app VPCs | Inbound/outbound endpoints + rules in the Networking Account, shared via RAM |
| **PHZs in Member Accounts** | Each app team owns its PHZ, associated to its own VPC and the central VPC | Networking Account owns the endpoints and forwarding rules, shared via RAM |

### Scaling pain these setups hit

- Onboarding each new account/VPC needs coordinated manual steps (associate
  PHZs, RAM-share rules, apply per-environment) — cumbersome and error-prone at
  scale.
- With PHZs in member accounts, routing internal AWS queries through Resolver
  endpoints adds cost and scaling pressure as query volume grows.

## Migration scenario 1 — PHZs in the Networking Account

**Build the base setup (no impact to existing resolution):**

1. Create Profiles in the Networking Account (example: `DevApp`, `ProdApp`).
2. Associate PHZ-1 and PHZ-2 with `DevApp`; PHZ-3 with `ProdApp`.
3. RAM-share `DevApp` with the Dev accounts and `ProdApp` with the Prod account.
4. Associate the application-account VPCs with the shared Profiles.
5. Associate the Networking Account's Resolver rule with both Profiles.

Wait until all VPC, PHZ, and rule associations show **completed** before cutting
over.

**Cut over (one account at a time):**

1. Remove the direct PHZ ↔ VPC association in the app account (now served by the
   Profile).
2. Remove the direct Resolver-rule ↔ VPC association for on-prem resolution;
   once satisfied, you may stop RAM-sharing that rule.

Repeat for each remaining account.

## Migration scenario 2 — PHZs in Member Accounts

**Build the base setup:**

1. Create Profiles in the Networking Account (example: `DevApp`, `ProdApp`, plus
   a `Networking` profile for on-prem connectivity).
2. Associate the Networking Account VPC with the `Networking` profile.
3. Associate both Resolver rules (forward to on-prem DNS; and to the inbound
   endpoint) with `DevApp` and `ProdApp`.
4. RAM-share the Profiles with the application accounts.
5. Associate application-account VPCs with the shared Profiles (`DevApp` → App1/App2 Dev; `ProdApp` → App1 Prod).
6. Associate the application-account PHZs with the shared Profiles.

Wait for all associations to reach **completed**.

**Cut over (one account at a time):**

1. Remove the Resolver rule-2 ↔ VPC association for intra-AWS resolution; later,
   stop RAM-sharing it if satisfied.
2. Remove the PHZ-1 ↔ Networking-VPC association (inbound-endpoint path).
3. Remove the Resolver rule-1 ↔ VPC association for on-prem resolution via the
   outbound endpoint; later, stop RAM-sharing it.

Repeat for each remaining account.

## Post-migration benefits

- **Separation of concerns:** admins own the Profile structure and sharing; app
  teams create their PHZs and attach to the right Profile. Less coordination,
  more agility.
- **Lower cost:** taking internal AWS DNS traffic off the Resolver-endpoint path
  removes endpoint charges for that traffic.

## Considerations

!!! warning "Test first"
    Migration is seamless and needs no downtime, but validate it in a **test
    environment** before touching production.

- A VPC can be associated with **only one Profile at a time** — encapsulate the
  DNS needs of all components in a VPC into a single Profile.
- Profile–VPC associations are billed **hourly per association** (see the Route 53
  pricing page). Don't go too granular, or the association count (and cost)
  balloons.
- When a VPC has both a direct PHZ association and a Profile, Route 53 Resolver
  uses the **direct PHZ association first**.

---

*See also:* [Route 53 VPC Resolver — Best Practices](route53-resolver-best-practices.md)
· [Public-Facing VPC Lattice (Multi-Account)](vpc-lattice-multi-account.md)
· [Split-Horizon DNS Forwarding](../system/split-horizon-dns-forwarding.md)
