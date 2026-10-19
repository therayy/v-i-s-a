# CSP-Hosted TFE Architecture (Visa)

## 1. The problem

Visa wants Terraform Enterprise (TFE) to be a Tier-0 platform. Today it cannot be, because the PostgreSQL failover design is a custom setup (PostgreSQL with PgBouncer) that HashiCorp Engineering does not support.

The question this document answers:

> Can TFE run in a public cloud (AWS, Azure, or GCP) on supported managed services, so that Visa gets the resiliency it needs without a custom, unsupported database design?

Short answer: likely yes, but only after Visa's recovery targets are checked against the chosen cloud's managed PostgreSQL service.

## 2. The target architecture

```
                     Users / CI / API
                            |
                  [ Load balancer (HTTPS) ]
                    /                  \
         [ TFE node A ]           [ TFE node B ]      (active-active, 2+ availability zones)
                    \                  /
                     \                /
    +------------------------------------------------+
    |  Managed PostgreSQL                            |  <- TFE metadata
    |  (multi-AZ, automatic failover, backups)       |
    +------------------------------------------------+
    |  Object storage (S3 / Azure Blob / GCS)        |  <- state files, plans, logs
    +------------------------------------------------+
    |  Managed Redis                                 |  <- cache and coordination
    +------------------------------------------------+

    [ TFE agents ] inside Visa's on-prem network
          |  outbound HTTPS only
          +----------> TFE (cloud)
          |
          +----------> on-prem targets (VMware, networks, APIs)
```

### Components in plain terms

| Component | What it does | Why it matters |
|---|---|---|
| Load balancer | Sends each request to a healthy TFE node | One node failing does not stop TFE |
| TFE nodes (2 or more, different zones) | Run the Terraform application | Stateless, so a broken node can be replaced without data loss |
| Managed PostgreSQL | Stores all TFE metadata (workspaces, runs, users, policies) | The cloud provider handles replication, backups, and failover, replacing Visa's custom setup |
| Object storage | Stores state files, plans, and logs | Durable across zones by default |
| Managed Redis | Cache and coordination | Replication is not a concern, per the ticket |
| TFE agents (on-prem) | Run Terraform against on-prem targets | Connect outward over HTTPS, so the cloud side never needs inbound access into Visa's network |

## 3. Why this supports Tier-0

- **Database failover is supported.** The managed service does failover, not Visa's own scripts.
- **The application survives a node or zone failure.** Traffic moves to the remaining nodes.
- **No critical part sits on one VM.** Every layer has a replacement path.
- **Visa's cloud-first direction is met.** The design uses the public cloud for hosting, not just as a backup site.

## 4. What this design does not cover

- **Full region outage.** Multi-zone protects against a zone failure only. A region-wide outage needs a second region or a tested restore procedure, with its own RTO and RPO.
- **Operations stay with Visa.** Upgrades, patching, monitoring, security, and networking still belong to Visa's team.
- **The HA PostgreSQL integration is beta.** Per the ticket, the newer HA cluster integration is documented as beta. Visa's exact design must be checked with HashiCorp Engineering before it is called supported.
- **Cloud alone does not make TFE Tier-0.** Tier-0 status also depends on written RTO and RPO, tested failover, and runbooks.

## 5. Database options (the core decision)

| Option | Failover | Fit |
|---|---|---|
| Amazon RDS for PostgreSQL (Multi-AZ) | Automatic standby promotion in another zone | Simple, supported, a good first choice on AWS |
| Amazon Aurora PostgreSQL | Fast automatic failover, replicas in several zones | Lower failover time; a stronger option if Visa's RTO is short |
| Azure Database for PostgreSQL (zone-redundant) | Automatic failover to a standby zone | Equivalent option on Azure |
| Google Cloud SQL for PostgreSQL (regional HA) | Automatic failover to a standby zone | Equivalent option on GCP |

Connectivity to the database (endpoints, DNS, connection pooling) is customer-managed. If Visa still wants PgBouncer, it is run as a supported component in front of the managed database, not as a custom failover system.

## 6. Migration path (on-prem to cloud)

1. **Build the cloud environment.** Create the nodes, load balancer, managed PostgreSQL, object storage, and Redis with infrastructure as code.
2. **Install and test a new TFE instance.** Use the same version as on-prem, or a version the migration guide supports.
3. **Back up the current on-prem TFE.** Use TFE's supported backup, which covers the PostgreSQL database and object storage.
4. **Restore into the cloud environment.** Test it with a copy of production data first.
5. **Check it.** Confirm runs, state access, SSO, and VCS connections.
6. **Move agents.** Point agents at the cloud TFE. Keep on-prem TFE as a fallback until the cutover is confirmed.
7. **Cut over DNS.** Switch the TFE hostname to the cloud load balancer.
8. **Run both in parallel for a period**, then retire the on-prem VMs.

Plan the migration with HashiCorp's migration documentation and a support engagement. Test the restore at least once before the cutover date.

## 7. Managing on-prem infrastructure from the cloud

Visa wants the cloud TFE to keep managing on-prem systems. The safe pattern is:

- **Agents live on-prem, close to the targets.** Each agent pool runs in Visa's network.
- **Agents connect outward only.** They open HTTPS connections to the cloud TFE. No inbound firewall holes are needed for TFE.
- **Agents reach the on-prem APIs they need.** Example: the vSphere API for VMware, or network device APIs.
- **Separate agent pools by zone or purpose.** This limits the blast radius and matches Visa's security boundaries.
- **Keep the agents highly available too.** Run at least two agents per pool so one host failure does not stop deployments.

## 8. Comparing the options

| | On-prem TFE (today) | Cloud-hosted TFE (proposed) | HCP Terraform (SaaS) |
|---|---|---|---|
| Who runs the app | Visa | Visa | HashiCorp |
| Database failover | Custom (not supported) | Managed service (supported) | HashiCorp |
| State storage control | Visa | Visa (their own bucket) | Raised as a concern by Visa |
| BYOK / encryption control | Visa | Visa | Raised as a concern by Visa |
| Fit with cloud-first strategy | Low | High | Medium |
| Operational effort for Visa | High | Medium | Low |

## 9. Discovery questions for Visa

**Recovery targets**
1. What RTO and RPO does Visa need for TFE as a Tier-0 platform?
2. Does Visa need survival of a single zone, or a full region?
3. What is the failover behavior required for the database?

**Cloud and platform**
4. Which cloud does Visa prefer, and is there an existing landing zone?
5. Are there cloud provider limits or approved services Visa must follow?
6. Who will run TFE day to day, and what skills do they have?

**Data and security**
7. Are BYOK (customer-managed keys) and state-storage location requirements firm?
8. What compliance or data residency rules apply to state and plan data?
9. What SSO, audit, and logging systems must TFE connect to?

**Network and agents**
10. How do on-prem networks reach the cloud? Is there a private link or VPN?
11. How many agents and agent pools are planned, and where do they sit?
12. Which on-prem targets must agents reach?

**Scale and current state**
13. How many workspaces, runs, and users does TFE handle today?
14. What version runs on-prem, and how often is it upgraded?
15. What is the current PostgreSQL and PgBouncer setup, and what failed in past incidents?

**Decision context**
16. What does Visa compare Terraform against, and what is env0 being evaluated for?
17. What would make Visa choose HCP Terraform or another option instead?

## 10. Next steps

1. Get Visa's RTO and RPO numbers (questions 1 to 3).
2. Confirm the target cloud (question 4).
3. Check the chosen managed PostgreSQL service against those numbers.
4. Confirm the HA PostgreSQL integration status with HashiCorp Engineering.
5. Decide on single-region or cross-region DR.
6. Agree on the migration plan and a test restore date.