# Infrastructure Baseline — Reference Document

**Languages:** [Español](../es/README.md) · [English](../en/README.md) · [Português](../pt/README.md)

> ### ⚠️ Important notice — read before using
>
> These baselines are a **reference example**, not a production-ready configuration. Every
> infrastructure has different requirements: scale, budget, regulatory compliance, geographic
> region and team maturity. Treat this material as a **guide** and **adapt it** to your case.
>
> Before implementing any decision described here, review it with a professional who
> understands your context. **Delzo.cloud takes no responsibility** for implementations based
> on this document without proper review.

---

> A set of foundational decisions worth making **before** provisioning the first resource in
> the cloud. Defining these early prevents later problems: fuzzy billing, overlapping network
> ranges, DNS managed by hand, and no traceability over who approved each change.
>
> This document applies to **AWS**, **Azure** and **GCP**. Since each provider models the
> network and account organization differently, every section covers all three.

---

## 1. Separation by account / subscription / project

**Principle:** an isolated billing unit per **project × environment**. `prod` and `dev`
environments never share an account. This gives clean billing, isolates the blast radius of a
mistake, and enables independent permissions per environment.

| Cloud | Isolation unit                     | Root container                      |
|-------|------------------------------------|-------------------------------------|
| AWS   | **Account** (one per proj×env)     | **Organization** with a Management (Root) Account |
| Azure | **Subscription** (one per proj×env) | **Management Groups** under the Tenant |
| GCP   | **Project** (one per proj×env)     | **Organization** + **Folders**      |

### AWS
```
Management (Root) Account  ── billing + Organizations only, no workloads
├── OU: Security       → account: log-archive, account: audit
├── OU: Infrastructure → account: network (hub), account: shared-services
└── OU: Workloads
    ├── OU: Prod       → account: <org>-<project>-prod
    ├── OU: Staging    → account: <org>-<project>-staging
    └── OU: Dev        → account: <org>-<project>-dev
```
- The Management Account **hosts no resources**: it is limited to AWS Organizations,
  consolidated billing and **SCPs** (Service Control Policies).
- **SCPs** per OU act as guardrails (for example: restricting which regions are enabled or
  preventing CloudTrail from being disabled).

### Azure
```
Tenant (Entra ID)
└── Management Group: root
    ├── MG: Platform      → subscription: connectivity, subscription: management
    └── MG: Landing Zones
        ├── MG: Prod      → subscription: <org>-<project>-prod
        ├── MG: Staging   → subscription: <org>-<project>-staging
        └── MG: Dev       → subscription: <org>-<project>-dev
```
- **Azure Policy** applied on the Management Groups fulfils the guardrail role, equivalent to
  AWS SCPs.

### GCP
```
Organization
├── Folder: prod     → project: <org>-<project>-prod
├── Folder: staging  → project: <org>-<project>-staging
├── Folder: dev      → project: <org>-<project>-dev
└── Folder: shared   → project: <org>-network-hub, project: <org>-logging
```
- **Organization Policies** per Folder act as guardrails.
- Billing is centralized in a **Billing Account** linked to all projects.

---

## 2. IP / CIDR addressing

**Principle:** assign a **/20** block (4,096 addresses) per **project × environment**, drawn
from an addressing plan defined in advance. Ranges are not left to the provider default:
otherwise, when interconnecting networks (peering / VPN) the ranges overlap and the connection
becomes impossible.

### Why /20 and not /16

A **/16** offers ~65,000 addresses. In practice, an environment rarely consumes a significant
fraction of that, so reserving a /16 per environment **wastes address space** and exhausts the
organization's global plan sooner.

A **/20** (4,096 addresses) is sufficient and efficient. The breakdown below shows that a
single /20 accommodates a multi-zone Kubernetes cluster, its load balancers and its databases,
and still leaves addresses free:

| Use                         | Prefix | Addresses | Comment |
|-----------------------------|--------|-----------|---------|
| Kubernetes nodes + pods     | **/22 per zone** | ~1,024 per zone | With 3 zones: comfortable for most workloads, even more with Fargate/serverless |
| Load Balancers              | **/24 per zone** | 256 per zone | |
| Databases                   | **/26 per zone** | 64 per zone | Isolated, no route to the internet |
| Reserved / growth           | remainder | —        | Free space remains within the same /20 |

> **Capacity note:** ~1,000 addresses per zone for nodes and pods covers the vast majority of
> scenarios. Only a large-scale application concentrated in a single cluster would need more;
> in that case you assign another /20 to that project, instead of oversizing every environment
> with a /16.

### Example partition of a /20 (`10.10.0.0/20`)

```
10.10.0.0/20  (4,096 addresses)
├── Kubernetes (nodes + pods)         /22 per zone
│   ├── zone a   10.10.0.0/22
│   ├── zone b   10.10.4.0/22
│   └── zone c   10.10.8.0/22
├── Load Balancers                    /24 per zone
│   ├── zone a   10.10.12.0/24
│   ├── zone b   10.10.13.0/24
│   └── zone c   10.10.14.0/24
├── Databases                         /26 per zone
│   ├── zone a   10.10.15.0/26
│   ├── zone b   10.10.15.64/26
│   └── zone c   10.10.15.128/26
└── Free / growth   10.10.15.192/26
```

### How it materializes in each cloud

Each provider models the network differently. The same /20 plan applies as follows:

**AWS — regional network, subnets per Availability Zone.**
The mapping is direct: each `/22`, `/24` and `/26` is a subnet in a specific AZ.
- VPC with CIDR `10.10.0.0/20`.
- 3 private `/22` subnets (EKS nodes/pods), 3 public `/24` subnets (LB/NLB), 3 isolated `/26`
  subnets (RDS, no internet gateway).
- With VPC-CNI each pod gets an IP from the subnet; if higher pod density is required, use a
  secondary CIDR in CGNAT space (`100.64.0.0/10`), without touching the main /20.

**Azure — regional network; the subnet spans the region's zones.**
In Azure the zone is a property of the resource, not the subnet. The VNet's /20 is split
**by tier** (not by zone):
- VNet `10.10.0.0/20`.
- `snet-aks` /22, `snet-lb` /24, `snet-db` /26 (with *delegation* to the managed database).
- With **Azure CNI Overlay** pods use their own overlay range and don't consume VNet IPs.

**GCP — the VPC is global; the subnet is regional (spans zones).**
The subnet covers all zones in its region, so the /20 is also split **by tier**:
- Global VPC; regional subnet `10.10.0.0/22` for nodes.
- GKE uses native **alias IP / secondary ranges**: a secondary `pods` range and a secondary
  `services` range, defined on the same subnet.
- The internal LB uses a *proxy-only subnet* `/24`; managed databases connect via Private
  Service Access `/26`.

**Key difference to remember:** in AWS the subnet lives in **a single AZ** (hence the
per-zone partition); in Azure and GCP the subnet **spans several zones** of the region, so the
same /20 is split by service tier. In addition, on GCP the VPC is **global** and supports
resources in several regions without peering.

---

## 3. Centralized domains and DNS

**Principle:** the domain **registration** and **delegation** are managed in a single place (a
dedicated DNS account or project). Domains are not registered in personal accounts.

| Cloud     | Managed DNS service |
|-----------|---------------------|
| AWS       | Route 53 |
| Azure     | Azure DNS |
| GCP       | Cloud DNS |
| Multi/CDN | Cloudflare (recommended as a central layer with WAF/CDN) |

### Rules for records
- **Avoid `A` records with a manually fixed IP.** A fixed IP becomes stale when the resource
  rotates, causing an outage.
- Prefer **`CNAME`** pointing to the resource's hostname (the LB or CDN manages the IPs).
- On **AWS**, for the domain apex (which by RFC does not allow CNAME) use Route 53 **Alias
  records** toward ELB / CloudFront / S3 (no per-query cost).
- The apex equivalent in **Azure DNS** is the **Alias record**; on **GCP/Cloudflare** it is
  resolved with **CNAME flattening**.

### Delegation
- Per-environment subdomains (`staging.example.com`, `dev.example.com`) are delegated with
  **`NS`** records toward separate zones, so that production does not depend on changes made in
  lower environments.

---

## 4. Naming convention

**Principle:** the name must indicate **what it is and where it is** without needing to open
the console.

```
<environment>-<service>-<region>-<resource-type>
```

Examples:
```
prod-frontend-us-east-1-eks
prod-api-us-east-1-rds
staging-api-eastus-aks
dev-worker-southamerica-east1-gke
prod-network-us-east-1-vpc
```

| Segment       | Example values                                        |
|---------------|-------------------------------------------------------|
| environment   | `prod` · `staging` · `dev`                            |
| service       | `frontend` · `api` · `worker` · `network`            |
| region        | `us-east-1` (AWS) · `eastus` (Azure) · `us-east1` (GCP) |
| resource-type | `eks`/`aks`/`gke` · `vpc` · `rds` · `lb` · `bucket`  |

- **Lowercase, hyphen-separated.** No uppercase, spaces or underscores.
- Naming is complemented by mandatory **tags/labels** (see section 10): the name is for human
  reading; tags enable billing and automation.

---

## 5. SSO / Identity

**Principle:** access to the clouds and to SaaS applications is done via **SSO** from a single
**Identity Provider (IdP)**. No local per-service credentials are created.

| IdP                    | Suitable context |
|------------------------|------------------|
| **Okta**               | Broad ecosystem of third-party SaaS |
| **Google Workspace**   | Organizations already based on Google Workspace |
| **Microsoft Entra ID** | Azure / Microsoft 365-centric environments |

- The IdP is federated to the three clouds via **SAML/OIDC** (AWS IAM Identity Center, Entra ID
  for Azure, Workforce Identity for GCP).
- **MFA mandatory** for all accounts.
- **Machine identities** (CI/CD, workloads) don't use SSO: they use **OIDC-federated roles**,
  avoiding long-lived static credentials.
- **Onboarding and offboarding of people** is managed in the IdP: disabling a user
  simultaneously revokes their access to all systems.

---

## 6. Code control — repositories, approvals and CODEOWNERS

**Principle:** no change reaches the main branch without review. The Git history is the audit
trail.

- Standardize on **GitHub** or **GitLab** (one of the two).
- **Protection of the `main` branch:**
  - No direct push.
  - Pull/Merge Request required, with **1–2 approvals** minimum.
  - Green CI checks (lint, tests, build) required to merge.
  - Commit signing (recommended).
- **CODEOWNERS:** assigns automatic reviewers per path.
  ```
  # .github/CODEOWNERS  (or CODEOWNERS in GitLab)
  /app/        @org/backend
  /web/        @org/frontend
  /infra/      @org/platform
  *.tf         @org/platform
  ```
- **Repository names:** `<org>-<project>[-<component>]`, lowercase and hyphenated.
- **Branch names:** `feat/…`, `fix/…`, `chore/…`, `hotfix/…`.
- **Infrastructure as code** lives in its own repository, with the `plan` visible in the PR
  before applying.

---

## 7. Corporate password manager

**Principle:** credentials are not shared via messaging, spreadsheets or `.env` files.

- Team password manager: **1Password**, **Bitwarden** or **Vaultwarden** (self-hosted).
- **Vaults per team or project** with segmented access.
- The password manager stores **human secrets** (SaaS logins, cards). **Machine secrets** (API
  keys, database passwords) are stored in the **cloud secrets manager** (AWS Secrets Manager /
  Azure Key Vault / GCP Secret Manager), never in the password manager.
- Ideally, the password manager itself is accessed via **SSO** (section 5).

---

## 8. Ticketing system

**Principle:** every request, incident or change is recorded. What is not in a ticket did not
happen.

- **Internal management:** Jira, Linear, or GitHub/GitLab Issues for technical work.
- **External support:** a helpdesk that turns email into tickets (Zammad, Freshdesk or an
  equivalent option).
- **Change management:** each production change is documented in a ticket with an owner,
  description, date and rollback plan, linked to the Git PR.
- **Per-project prefixes** for traceability (for example `INFRA-`, `SEC-`, `OPS-`).

---

## 9. How the pieces fit together

```
        IdP (Okta / Google / Entra) ──SSO/MFA──┐
                                                ▼
   ┌────────── Root / Org / Tenant  (billing + guardrails) ───────────┐
   │  DNS Hub (Route53/Cloudflare)   Secrets Manager   Log Archive     │
   └──────────────────────────────────────────────────────────────────┘
        │  proj×env = account / subscription / project,  /20 CIDR each
        ▼
   prod-<project>    staging-<project>    dev-<project>    ...
        │
   Git (PR + CODEOWNERS) → CI/CD (OIDC, keyless) → IaC → resources with naming + tags
        │
   Ticketing system (every change is recorded)
```

---

## 10. Complementary baseline components

Elements that are often omitted in the initial definition and worth incorporating from the
start:

1. **Mandatory tags/labels strategy** — `env`, `project`, `owner`, `cost-center`,
   `managed-by`. Without them you cannot allocate billing or automate.
2. **Infrastructure as code** — Terraform/OpenTofu or Pulumi. Production changes are not made
   through the console; the console remains read-only access.
3. **Centralized logging and audit** — CloudTrail / Azure Activity Log / Cloud Audit Logs sent
   to an **immutable log-archive account**, not modifiable even by administrators.
4. **Backup and Disaster Recovery** — 3-2-1 policy, cross-account/region copies and **RTO/RPO**
   defined in writing. A backup not tested via restore does not count as a backup.
5. **Budgets and cost alerts** — AWS Budgets / Azure Cost Alerts / GCP Budgets with notices at
   thresholds (50/80/100%).
6. **Break-glass / emergency account** — a last-resort access identity with physical MFA, for
   the case of IdP unavailability. Documented and audited on each use.
7. **Security baseline** — threat detection (GuardDuty / Defender / Security Command Center),
   encryption at rest by default, private storage by default, and vulnerability scanning in
   the pipeline.
8. **Device management (MDM)** — machines with encrypted disks and automatic lock.
9. **Incident response runbook** — owners, communication channels and documentation method
   defined ahead of time.
10. **Vendor / SaaS registry** — an inventory of tools with owner, cost and renewal date, to
    control SaaS sprawl.
11. **Data classification and compliance** — identification of sensitive data (PII, cardholder
    data → PCI DSS) and its location.
12. **Sandbox environment** — a disposable account for experimentation, with an aggressive
    budget cap and automatic teardown.

---

*Reference document — version 1. Published by [Delzo.cloud](https://delzo.cloud) under the
[CC BY 4.0](../LICENSE) license. Remember: this is an example guide, adapt it to your own
infrastructure.*