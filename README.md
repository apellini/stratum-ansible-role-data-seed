# apellini.data_seed

Ansible role — **STRATUM dev Phase 3 data seeding**.

Restores Qdrant vector KB and MLflow model artifacts from the GCS bootstrap cache
(`gs://<cache_bucket>/qdrant/` and `gs://<cache_bucket>/mlflow/`) at session boot.
Creates the Ceph RGW user + `mlflow-s3-creds` Kubernetes secret as a prerequisite
for MLflow's S3 artifact store.

Soft-skips both restores when no GCS data exists (first run — not an error).

## Decision

D-INFRA-29 — GCS as the authoritative cross-session store for Qdrant snapshots and
MLflow artifacts. See `apellini/stratum-infra` for the full decision record.

## Requirements

- Single-node k3s cluster with Linkerd (from `apellini.k3s` + `apellini.linkerd`)
- Rook-Ceph operator + cluster with a `CephObjectStore` named `ceph-objectstore`
  (from `apellini.helm_workloads` — rook-ceph + rook-ceph-cluster charts)
- Qdrant deployed as `qdrant` release in `stratum-data` namespace
- MLflow deployed as `mlflow` release in `stratum-ops` namespace, configured with:
  - `postgresql.enabled: true` (bitnami subchart; password auto-generated, no plaintext)
  - `artifactRoot.s3.existingSecret.name: mlflow-s3-creds`
- `kubernetes.core` collection installed on the control node
- GCE compute service account attached to the k3s VM with
  `roles/storage.objectViewer` on the GCS cache bucket (for GCS reads)
- `google-cloud-sdk` and `awscli` (installed by this role)

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `data_seed_cache_bucket` | `""` | GCS cache bucket name (required; set by harness.sh) |
| `data_seed_rgw_endpoint` | `http://rook-ceph-rgw-ceph-objectstore.rook-ceph.svc:7480` | Ceph RGW S3 endpoint |
| `data_seed_rgw_object_store` | `ceph-objectstore` | CephObjectStore name |
| `data_seed_rgw_user` | `mlflow-user` | CephObjectStoreUser name |
| `data_seed_mlflow_bucket` | `mlflow` | S3 bucket name for MLflow artifacts |
| `data_seed_mlflow_namespace` | `stratum-ops` | Namespace where MLflow is deployed |
| `data_seed_data_namespace` | `stratum-data` | Namespace where Qdrant is deployed |
| `data_seed_qdrant_collection` | `stratum-kb` | Qdrant collection to restore |
| `data_seed_qdrant_snapshot_name` | `stratum-kb.snapshot` | Qdrant snapshot filename in GCS |

## Task Flow

```
install prereqs (gcloud SDK, awscli)
  → wait for Qdrant ready (stratum-data)
  → wait for MLflow-embedded PostgreSQL ready (stratum-ops/mlflow-postgresql)
  → wait for RGW ready
  → create CephObjectStoreUser (mlflow-user)
  → wait for RGW user secret (Rook creates it asynchronously)
  → create mlflow-s3-creds secret in stratum-ops
  → create mlflow S3 bucket
  → wait for MLflow ready (secret now exists → pod can start)
  → Qdrant restore (GCS → pod → REST upload) or first-run skip
  → MLflow restore (GCS → tmpdir → Ceph S3 sync) or first-run skip
```

## Teardown counterpart

`backup_phase3_data` in `dev/harness.sh` (stratum-infra) exports data back to GCS
before teardown: Qdrant snapshot upload + MLflow Ceph S3 → GCS rsync.

## Example

```yaml
- name: Seed Qdrant and MLflow from GCS cache
  hosts: k3s
  become: true
  roles:
    - role: apellini.data_seed
      vars:
        data_seed_cache_bucket: "{{ lookup('env', 'STRATUM_GCS_CACHE_BUCKET') }}"
```
