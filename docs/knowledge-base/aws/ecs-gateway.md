---
tags:
  - aws
  - ecs
  - networking
  - ingress
  - load-balancing
  - containers
---

# Amazon ECS Gateway

**Amazon ECS Gateway** simplifies ingress for containerized apps. Instead of
wiring an Application Load Balancer (ALB), listeners, target groups, security
groups, and certificates for every service, you define ingress **once** on a
gateway and share that infrastructure across many services. Each connected
service gets a unique HTTPS hostname.

!!! info "Source"
    Condensed from the AWS docs
    [Amazon ECS Gateway overview](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/gateway-overview.html).
    Content was rephrased for compliance with licensing restrictions. Check the
    AWS documentation for authoritative, current details.

## How it works — two personas

ECS Gateway separates **infrastructure management** from **application
deployment**:

- **Platform teams own the gateway.** They create and configure it and
  associate it with ECS clusters. The gateway config applies to every connected
  service and spans:
    - **Networking** — internet-facing vs. VPC-only, which subnets the load
      balancers use.
    - **Security** — the inbound security group, and which AWS WAF web ACL
      applies.
    - **Operations** — where access logs go, how much load balancer capacity to
      allow, and which request header marks test traffic for progressive
      deployments.
- **Application teams opt in.** They add a **gateway connector** when creating
  or updating a service, naming the container port that receives traffic (plus
  optional access-type filter and health checks).

### Three steps to ingress

1. **Create a gateway** (platform team) with its networking/security config.
2. **Associate the gateway with clusters** to make it available to services
   there.
3. **Connect services** — each opts in with a connector naming its traffic port.

!!! note "Lazy provisioning"
    A gateway provisions **nothing** when created or associated with a cluster.
    When the **first service connects**, ECS creates/reuses shared resources,
    adds the per-service resources, and creates a gateway route to a unique
    HTTPS URL.

## Key concepts

| Concept | Description |
| :--- | :--- |
| **Gateway** | Account-level resource (own ARN) defining ingress config and managing the shared infrastructure. Names unique per account + Region. |
| **Gateway–cluster association** | Many-to-many link making a gateway available to a cluster's services. One gateway → many clusters; one cluster → many gateways. |
| **Gateway connector** | Opt-in parameter on a service connecting it to all compatible gateways in its cluster. Names the container port; optional access type + health checks. |
| **Gateway route** | Per-connection resource (own ARN) created when a service connects, deleted when it disconnects. Reports status, the service URL, and service-specific resources. |

## Infrastructure ECS manages

| Type | Resources | Notes |
| :--- | :--- | :--- |
| **Shared** | ALB, listener, LB security group | Managed by the gateway and shared across services. |
| **Service-specific** | ACM certificate, DNS hostname, listener rule, target group, service security group | Created per gateway route to direct traffic to one service. |

- Routing is **host-based** off a shared ALB. Each service gets a generated
  hostname like `https://<unique-id>.ecs.<region>.on.aws` (domain varies by
  partition).
- **Up to 25 services share one ALB**; each keeps its own hostname, certificate,
  and listener rules. ECS provisions another ALB when the current one is full.
- Managed resources are tagged `AmazonECSManaged=true` and
  `aws:ecs:gateway-arn`; per-route resources also carry
  `aws:ecs:gateway-route-arn` — use these tags to find them in the ELB, EC2, and
  ACM consoles.

## Availability & pricing

- **No extra charge** for ECS Gateway itself — you pay only for the underlying
  resources (ALB, CloudWatch logs/metrics, data transfer).
- Available in **all commercial Regions**.
- Supports services using the **`awsvpc` network mode** on **all launch types**
  (EC2, Fargate, ECS Managed Instances).
- Manageable via console, AWS CLI, SDKs, and CloudFormation.

## Service limits

| Limit | Value |
| :--- | :--- |
| Gateway–cluster associations per cluster | 5 |
| Clusters per gateway | 10,000 |
| Gateway connectors per service | 1 |
| Services per ALB | 25 |
| ALBs per gateway | 200 |

Underlying resources also have their own quotas (check ALB quotas).

## Setup runbook — AWS CLI

End-to-end: platform team creates the gateway and associates it with a cluster,
then an app team connects a service. Requires a recent AWS CLI v2 (ECS Gateway
commands landed in v2.38+).

!!! info "Verify your CLI first"
    ```bash
    aws --version          # expect aws-cli/2.38.x or newer
    aws ecs create-gateway help >/dev/null && echo "create-gateway available"
    ```

### Prerequisites

- An ECS **cluster** in the target Region.
- One or more **subnets** for the managed ALBs (defaults to the default VPC's
  subnets if omitted).
- An **IAM infrastructure role** the gateway assumes to create/manage ingress
  resources on your behalf. You must have `iam:PassRole` for it.

### Step 0 — Set variables

```bash
export AWS_REGION=ap-southeast-1
export CLUSTER=my-cluster
export GATEWAY_NAME=platform-ingress
export INFRA_ROLE_ARN=arn:aws:iam::<account-id>:role/ecsGatewayInfrastructureRole
export SUBNET_A=subnet-aaaa1111
export SUBNET_B=subnet-bbbb2222      # at least two AZs for HA
export INBOUND_SG=sg-0123456789abcdef0
```

### Step 1 — Create the IAM infrastructure role (if you don't have one)

The gateway needs a role it can assume to provision ALBs, target groups, ACM
certs, etc. Trust it to the ECS service principal and attach the permissions
your org's policy allows.

```bash
cat > trust.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ecs.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
JSON

aws iam create-role \
  --role-name ecsGatewayInfrastructureRole \
  --assume-role-policy-document file://trust.json
```

!!! warning "Scope the role carefully"
    This role can create and delete load balancers, security groups, target
    groups, listener rules, and ACM certificates. Attach a least-privilege
    policy per your security baseline rather than a broad managed policy. Check
    the AWS docs for the current required permissions.

### Step 2 — Create the gateway

The `--configuration` is a **tagged union**; for host-based shared ALB ingress
set the `hostSharedAlb` variant. Put the config in a file to keep it readable.

```bash
cat > gateway-config.json <<JSON
{
  "hostSharedAlb": {
    "networkConfiguration": {
      "inboundSecurityGroup": "${INBOUND_SG}",
      "subnets": ["${SUBNET_A}", "${SUBNET_B}"],
      "ipAddressType": "dualstack"
    },
    "scheme": "internet-facing",
    "port": 443,
    "protocol": "HTTPS",
    "maxAlbCount": 10,
    "httpDesync": "defensive",
    "accessLogs": {
      "destination": { "s3": { "bucket": "my-alb-access-logs", "prefix": "ecs-gateway" } }
    }
  }
}
JSON

aws ecs create-gateway \
  --region "$AWS_REGION" \
  --name "$GATEWAY_NAME" \
  --infrastructure-role-arn "$INFRA_ROLE_ARN" \
  --configuration file://gateway-config.json \
  --deletion-protection \
  --tags key=team,value=platform
```

The response returns `gatewayArn` and `status: ACTIVE`. Capture the ARN:

```bash
export GATEWAY_ARN=$(aws ecs create-gateway ... --query gatewayArn --output text)
# or, if already created:
export GATEWAY_ARN=$(aws ecs list-gateways --region "$AWS_REGION" \
  --query "gateways[?name=='${GATEWAY_NAME}'].gatewayArn | [0]" --output text)
```

!!! note
    No ALB is created yet. A gateway provisions infrastructure only when the
    first service connects.

### Step 3 — Associate the gateway with the cluster

```bash
aws ecs associate-gateway-with-cluster \
  --region "$AWS_REGION" \
  --gateway-identifier "$GATEWAY_ARN" \
  --cluster "$CLUSTER"
```

A cluster can associate with up to 5 gateways. Nothing deploys until a service
in the cluster opts in.

### Step 4 — Connect a service (app team)

Add a **gateway connector** on `create-service` (or `update-service`), naming
the container port that receives traffic. Put it in a file:

```bash
cat > connector.json <<'JSON'
[
  {
    "portName": "web",
    "accessType": "internet-facing",
    "healthCheck": { "path": "/healthz", "protocol": "HTTP" }
  }
]
JSON
```

> The connector's `portName` must match a `name` on a `portMapping` in your task
> definition. Confirm the exact connector field names with
> `aws ecs create-service help` on your CLI version, since this is a new API.

```bash
aws ecs create-service \
  --region "$AWS_REGION" \
  --cluster "$CLUSTER" \
  --service-name web \
  --task-definition web:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[${SUBNET_A},${SUBNET_B}],securityGroups=[${INBOUND_SG}],assignPublicIp=DISABLED}" \
  --gateway-connectors file://connector.json
```

!!! warning
    `create-service`/`update-service` **fails** if the connector is set but no
    gateway is associated with the service's cluster. Associate the gateway
    (Step 3) first.

### Step 5 — Verify and get the URL

```bash
# Gateway status
aws ecs describe-gateways --region "$AWS_REGION" --gateways "$GATEWAY_ARN"

# The per-service route holds the generated HTTPS URL + status
aws ecs list-gateway-routes --region "$AWS_REGION" --gateway-identifier "$GATEWAY_ARN"
aws ecs describe-gateway-routes --region "$AWS_REGION" --gateway-route <route-arn> \
  --query 'gatewayRoutes[0].{status:status,url:url}'

# Find the managed ALB by tag
aws resourcegroupstaggingapi get-resources --region "$AWS_REGION" \
  --tag-filters Key=aws:ecs:gateway-arn,Values="$GATEWAY_ARN" \
  --resource-type-filters elasticloadbalancing
```

Once the route reports a completed status, `curl https://<id>.ecs.<region>.on.aws`.

### Teardown (reverse order)

```bash
# 1. Disconnect services (remove the connector) or delete them
aws ecs update-service --cluster "$CLUSTER" --service web --gateway-connectors '[]'
# 2. Disassociate the gateway from the cluster
aws ecs disassociate-gateway-from-cluster --gateway-identifier "$GATEWAY_ARN" --cluster "$CLUSTER"
# 3. Delete the gateway (disable deletion protection first if enabled)
aws ecs update-gateway --gateway-identifier "$GATEWAY_ARN" --no-deletion-protection
aws ecs delete-gateway --gateway-identifier "$GATEWAY_ARN"
```

!!! danger "Verify exact subcommands"
    Teardown subcommand names (`disassociate-gateway-from-cluster`,
    `delete-gateway`, `update-gateway`) are new — confirm with `aws ecs help`
    on your installed CLI version before running in production.

## Setup — Terraform

ECS Gateway is a new API, so a first-class `aws_ecs_gateway` resource **may not
exist yet** in the AWS provider. Two options:

### Option A — Native resource (once available)

Check your provider version first:

```bash
terraform providers
# then look for aws_ecs_gateway in the registry docs for your pinned version
```

If present, it will look roughly like this (confirm argument names against the
registry docs for your version — do not assume):

```hcl
# PSEUDOCODE — verify against the provider docs for your version
resource "aws_ecs_gateway" "platform_ingress" {
  name                    = "platform-ingress"
  infrastructure_role_arn = aws_iam_role.ecs_gateway.arn
  deletion_protection     = true

  configuration {
    host_shared_alb {
      scheme   = "internet-facing"
      port     = 443
      protocol = "HTTPS"

      network_configuration {
        inbound_security_group = aws_security_group.ingress.id
        subnets                = [aws_subnet.a.id, aws_subnet.b.id]
        ip_address_type        = "dualstack"
      }
    }
  }

  tags = { team = "platform" }
}
```

### Option B — Cloud Control API (works before a native resource ships)

Terraform's `awscc` provider or the `aws_cloudcontrolapi_resource` in the `aws`
provider can manage any CloudFormation-backed resource type — and ECS Gateway is
manageable via CloudFormation. Use `AWS::ECS::Gateway`:

```hcl
resource "aws_iam_role" "ecs_gateway" {
  name = "ecsGatewayInfrastructureRole"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ecs.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

# Attach a least-privilege policy to aws_iam_role.ecs_gateway here.

resource "aws_cloudcontrolapi_resource" "ecs_gateway" {
  type_name = "AWS::ECS::Gateway"

  desired_state = jsonencode({
    Name                  = "platform-ingress"
    InfrastructureRoleArn = aws_iam_role.ecs_gateway.arn
    DeletionProtection    = true
    Configuration = {
      HostSharedAlb = {
        Scheme   = "internet-facing"
        Port     = 443
        Protocol = "HTTPS"
        NetworkConfiguration = {
          InboundSecurityGroup = aws_security_group.ingress.id
          Subnets              = [aws_subnet.a.id, aws_subnet.b.id]
          IpAddressType        = "dualstack"
        }
      }
    }
  })
}
```

!!! warning "Confirm the CloudFormation schema"
    The exact `AWS::ECS::Gateway` property names (PascalCase in Cloud Control)
    must match the published CloudFormation resource schema for your Region.
    Pull the live schema before applying:
    ```bash
    aws cloudformation describe-type --type RESOURCE \
      --type-name AWS::ECS::Gateway --query 'Schema' --output text | jq .
    ```

### Association and the connector in Terraform

- **Gateway ↔ cluster association** and **gateway routes** are separate
  resources/APIs; manage them with the matching `awscc`/Cloud Control types
  (e.g. an association resource type) once their schemas are confirmed.
- The **gateway connector** lives on the **service**. If your provider's
  `aws_ecs_service` doesn't yet expose a `gateway_connector` block, manage that
  service via Cloud Control (`AWS::ECS::Service`) or the AWS CLI until the
  argument is added.

!!! tip "Pragmatic path today"
    Because the Terraform provider surface for a brand-new API lags the API
    itself, a common pattern is: provision the **IAM role, SGs, and subnets**
    with Terraform (stable resources), then create the **gateway, association,
    and connectors** with the AWS CLI (Step 1–4 above) or Cloud Control until
    native resources ship. Revisit once `aws_ecs_gateway` is GA in the provider.

## Considerations

!!! warning "Immutable settings"
    You **cannot change** a gateway's name, type, scheme, listener port/protocol,
    subnets, or IP address type after creation. To change them, create a new
    gateway and move services to it.

- **HTTPS only** — host-based routing over ALB, HTTPS on port 443. No other
  protocol.
- **`awsvpc` network mode required** — `bridge`, `host`, and `none` cannot
  connect.
- **Single account** — a gateway and its clusters must be in the same account.
- **No Express Mode** — serves only standard ECS services.
- **One connector per service** — but one connector can reach several compatible
  gateways.
- **No gateway-by-name selection** — a connector attaches to every *compatible*
  gateway in the cluster. Compatible = matches the access-type filter (if set),
  same VPC as the service, and covers every AZ the service runs in. If none
  match, the request fails.
- **No load-balancer consolidation** — ECS adds ALBs when full and removes empty
  ones, but never moves services between them, so partly filled ALBs can linger.
- **No drift repair** — if you modify/delete a managed resource yourself, ECS
  won't restore it; traffic can stop and later deployments/updates/cleanup can
  fail.

## When to reach for it

Good fit when platform teams want to hand application teams **ingress without
networking expertise**, share ALBs to cut cost, and keep all resources
**visible in your own account**. Less suitable when you need non-HTTPS
protocols, cross-account gateways, or fine-grained manual control of each ALB.

---

*See also:* [Gateway API on EKS](../k8s/gateway-api-eks.md)
· [AWS Load Balancer Controller](aws-load-balancer-controller.md)
· [Why We Migrated Ingress to Gateway API](../gcp/why-we-migrated-ingress-to-gateway-api.md)
