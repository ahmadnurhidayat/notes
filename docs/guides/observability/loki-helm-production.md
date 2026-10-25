---
tags:
  - observability
  - loki
  - grafana
  - helm
  - kubernetes
  - gcp
---

# Loki on Kubernetes with Helm (production values)

Running Loki in production on Kubernetes needs tuning of storage, ingestion,
and compaction. The default Helm values assume small-scale evaluation — ship
them as-is and you hit OOM kills on the ingester and ballooning storage.

This is the **SimpleScalable** setup with a **GCS** backend that I run on GKE.

!!! note "When to use SimpleScalable"
    It balances simplicity and performance for most workloads. Reach for the
    fully distributed mode only when read and write paths need to scale on
    very different curves.

## Storage backend (GCS)

```yaml
loki:
  storage:
    type: gcs
    bucketNames:
      chunks: my-loki-chunks
      ruler: my-loki-ruler
    gcs:
      chunkBufferSize: 0
```

Set a **bucket lifecycle rule** for cost control, and let Loki's own retention
handle logical deletion:

=== "Retention (Loki)"

    ```yaml
    loki:
      limits_config:
        retention_period: 744h   # 31 days
      compactor:
        retention_enabled: true
        delete_request_store: gcs
    ```

=== "Lifecycle (GCS)"

    ```json
    {
      "rule": [
        { "action": {"type": "Delete"},
          "condition": {"age": 40} }
      ]
    }
    ```

!!! warning
    Keep the GCS lifecycle age **longer** than Loki's `retention_period`.
    If the bucket deletes chunks before the compactor does, queries over the
    retention window return gaps.

## Ingestion limits

The single most common cause of ingester OOM is unbounded streams from label
explosion. Cap it:

```yaml
loki:
  limits_config:
    ingestion_rate_mb: 10
    ingestion_burst_size_mb: 20
    max_global_streams_per_user: 5000
    max_label_names_per_series: 15
```

## Metrics to watch

| Metric | Warning sign |
| :--- | :--- |
| `loki_ingester_streams_created_total` | Rapid growth → label explosion |
| `loki_ingester_chunk_entries` | Low count → tiny chunks, wasted I/O |
| `loki_compactor_blocks_compacted_total` | Falling behind → storage pressure |

## Grafana datasource

```yaml
apiVersion: 1
datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki-gateway.observability.svc:80
    jsonData:
      maxLines: 1000
```

---

*Ported from my blog post; the narrative version with the full `values.yaml`
lives on [docs.beyondyou.my.id](https://docs.beyondyou.my.id/docs/01-knowledge-base/observability/loki-helm-production).*
