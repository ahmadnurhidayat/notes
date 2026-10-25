---
date: 2026-10-25
tags:
  - kubernetes
  - networking
  - gateway-api
---

# Gateway API HTTPRoute needs a parentRef in the right namespace

When migrating from Ingress to Gateway API on GKE, an `HTTPRoute` silently
serves no traffic if its `parentRef` points at a `Gateway` in another
namespace without a matching `ReferenceGrant` — or without `allowedRoutes`
opening that namespace on the Gateway listener.

**Symptom:** route exists, `kubectl get httproute` looks healthy, but the
load balancer never programs the backend.

**Fix:** on the `Gateway` listener, set `allowedRoutes.namespaces.from` to
`Selector` (or `All`) so routes from the app namespace are accepted:

```yaml
listeners:
  - name: https
    protocol: HTTPS
    port: 443
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            gateway-access: "true"
```

Then label the app namespace with `gateway-access=true`. The route attaches
and the backend gets programmed within a reconcile cycle.

**Why it bites:** Ingress was namespace-local, so teams carry that mental
model into Gateway API — but Gateway is explicitly cross-namespace and
closed by default. That is a feature, not a bug.
