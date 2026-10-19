# CSP-Hosted TFE Architecture (Visa)

## Goal

Replace Visa's custom PostgreSQL/PgBouncer failover design with a supported architecture, so Terraform Enterprise (TFE) can meet Tier-0 resiliency requirements.

## Architecture overview

```
                    Users / CI / API
                           |
                 [ Load balancer (HTTPS) ]
                   /                \
        [ TFE node A ]          [ TFE node B ]     (active-active, 2+ AZs)
              \                    /
               \                  /
   +-----------------------------------------------+
   |  Managed PostgreSQL (multi-AZ, auto failover) |
   +-----------------------------------------------+
   |  Object storage (S3 / Blob / GCS)             |
   +-----------------------------------------------+
   |  Managed Redis                                |
   +-----------------------------------------------+

   [ TFE agents ] in Visa's on-prem network  ---HTTPS---> TFE (cloud)
        |
        +--> on-prem targets (VMware, networks, APIs)
```

## Components

| Component | Purpose | Notes |
|---|---|---|
| Load balancer | Spreads traffic across TFE nodes | Removes single-node failure |
| TFE nodes (2+, multiple AZs) | Terraform application | Stateless; replace without data loss |
| Managed PostgreSQL (e.g., RDS or Aurora) | All TFE metadata | Provider handles replication, backups, and zone failover |
| Object storage (S3 / Blob / GCS) | State files, plans, logs | Durable across zones by default |
| Managed Redis | Cache and coordination | Replication is not a concern for Visa |
| TFE agents (on-prem) | Run Terraform against on-prem targets | Outbound HTTPS only; no inbound access into Visa's network |

## Why this supports Tier-0

- Database failover is handled by a supported managed service, not custom scripts.
- The app tier survives a node or zone loss.
- No critical component lives on a single VM.

## Limits of this design

- **Region loss:** This design survives a zone failure. A full region outage needs a separate DR plan (second region or restore procedure) with a defined RTO and RPO.
- **Operational ownership:** Visa still owns upgrades, patching, networking, security, and monitoring.
- **Requirements must be validated:** Map Visa's RTO and RPO to the chosen cloud's managed PostgreSQL before committing.

## Next steps

1. Confirm Visa's RTO and RPO targets.
2. Confirm target cloud (AWS, Azure, or GCP).
3. Check the chosen managed PostgreSQL service against those targets.
4. Decide single-region or cross-region DR.
5. Validate the HA PostgreSQL integration status with HashiCorp Engineering (documented as beta).
