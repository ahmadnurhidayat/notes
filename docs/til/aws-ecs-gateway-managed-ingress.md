---
date: 2026-10-30
tags:
  - aws
  - ecs
  - networking
  - ingress
---

# AWS ECS Gateway gives you managed ingress without wiring an ALB per service

AWS shipped **Amazon ECS Gateway** — managed ingress for ECS. Instead of
building an ALB + listener + target group + security group + certificate for
every service, a platform team defines ingress **once** on a gateway, and each
service opts in with a **gateway connector** (naming its container port). ECS
then provisions the shared and per-service infrastructure and hands each service
a unique HTTPS hostname like `https://<id>.ecs.<region>.on.aws`.

**Why it's useful:** app teams get ingress with zero networking expertise, up to
**25 services share one ALB** (cost win), and every resource lives in your own
account (tagged `AmazonECSManaged=true`).

**Gotchas worth remembering:**

- HTTPS only, port 443 — no other protocol.
- Requires `awsvpc` network mode; single account only; no Express Mode.
- Several gateway settings (name, scheme, subnets, listener port/protocol, IP
  type) are **immutable** — to change them, make a new gateway and migrate.
- No ALB consolidation and no drift repair: don't hand-edit managed resources.

Full write-up: [Amazon ECS Gateway](../knowledge-base/aws/ecs-gateway.md).
