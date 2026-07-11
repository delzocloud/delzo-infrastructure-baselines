# Infrastructure Baseline — Documento de Referencia

**Idiomas:** [Español](../es/README.md) · [English](../en/README.md) · [Português](../pt/README.md)

> ### ⚠️ Aviso importante — leer antes de usar
>
> Estas baselines son un **ejemplo de referencia**, no una configuración lista para
> producción. Cada infraestructura tiene requisitos distintos: escala, presupuesto,
> cumplimiento normativo, región geográfica y madurez del equipo. Tomá este material como una
> **guía** y **adaptalo** a tu caso.
>
> Antes de implementar cualquier decisión aquí descrita, revisala con un profesional que
> conozca tu contexto. **Delzo.cloud no se responsabiliza** por implementaciones basadas en
> este documento sin la debida revisión.

---

> Conjunto de decisiones de base que conviene tomar **antes** de aprovisionar el primer
> recurso en la nube. Definir esto temprano evita, más adelante, problemas de facturación
> difusa, superposición de rangos de red, DNS gestionado manualmente y falta de trazabilidad
> sobre quién aprobó cada cambio.
>
> El documento aplica a **AWS**, **Azure** y **GCP**. Dado que cada proveedor modela la red y
> la organización de cuentas de forma distinta, cada sección incluye los tres.

---

## 1. Separación por cuenta / suscripción / proyecto

**Principio:** una unidad de facturación aislada por **proyecto × environment**. Los entornos
`prod` y `dev` nunca comparten cuenta. Esto permite un billing limpio, aísla el impacto de un
error, y habilita permisos independientes por entorno.

| Nube  | Unidad de aislamiento          | Contenedor raíz                     |
|-------|--------------------------------|-------------------------------------|
| AWS   | **Account** (una por proj×env) | **Organization** con Management (Root) Account |
| Azure | **Subscription** (una por proj×env) | **Management Groups** bajo el Tenant |
| GCP   | **Project** (uno por proj×env) | **Organization** + **Folders**      |

### AWS
```
Management (Root) Account  ── solo facturación + Organizations, sin workloads
├── OU: Security       → account: log-archive, account: audit
├── OU: Infrastructure → account: network (hub), account: shared-services
└── OU: Workloads
    ├── OU: Prod       → account: <org>-<project>-prod
    ├── OU: Staging    → account: <org>-<project>-staging
    └── OU: Dev        → account: <org>-<project>-dev
```
- La Management Account **no aloja recursos**: se limita a AWS Organizations, consolidated
  billing y **SCPs** (Service Control Policies).
- Las **SCPs** por OU actúan como guardrails (por ejemplo: restringir regiones habilitadas o
  impedir la desactivación de CloudTrail).

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
- **Azure Policy** aplicada sobre los Management Groups cumple el rol de guardrail,
  equivalente a las SCPs de AWS.

### GCP
```
Organization
├── Folder: prod     → project: <org>-<project>-prod
├── Folder: staging  → project: <org>-<project>-staging
├── Folder: dev      → project: <org>-<project>-dev
└── Folder: shared   → project: <org>-network-hub, project: <org>-logging
```
- **Organization Policies** por Folder actúan como guardrail.
- La facturación se centraliza en un **Billing Account** vinculado a todos los projects.

---

## 2. Direccionamiento IP / CIDR

**Principio:** asignar un bloque **/20** (4.096 direcciones) por **proyecto × environment**, a
partir de un plan de direccionamiento definido de antemano. Los rangos no se dejan librados al
default del proveedor: de lo contrario, al interconectar redes (peering / VPN) los rangos se
superponen y la conexión se vuelve inviable.

### Por qué /20 y no /16

Un **/16** ofrece ~65.000 direcciones. En la práctica, un entorno rara vez consume una
fracción significativa de eso, por lo que reservar un /16 por entorno **desperdicia espacio de
direccionamiento** y agota antes el plan global de la organización.

Un **/20** (4.096 direcciones) es suficiente y eficiente. El desglose siguiente muestra que un
único /20 acomoda un clúster de Kubernetes multi-zona, sus balanceadores y sus bases de datos,
y aún deja direcciones libres:

| Uso                         | Prefijo | Direcciones | Comentario |
|-----------------------------|---------|-------------|------------|
| Nodos + pods Kubernetes     | **/22 por zona** | ~1.024 por zona | Con 3 zonas: holgado para la mayoría de las cargas, incluso más con Fargate/serverless |
| Load Balancers              | **/24 por zona** | 256 por zona | |
| Bases de datos              | **/26 por zona** | 64 por zona | Aisladas, sin ruta a internet |
| Reservado / crecimiento     | resto   | —           | Queda espacio libre dentro del mismo /20 |

> **Nota de capacidad:** ~1.000 direcciones por zona para nodos y pods cubre la enorme mayoría
> de los escenarios. Solo una aplicación de gran escala concentrada en un único clúster
> requeriría más; en ese caso se asigna otro /20 a ese proyecto, en lugar de sobredimensionar
> todos los entornos con un /16.

### Ejemplo de partición de un /20 (`10.10.0.0/20`)

```
10.10.0.0/20  (4.096 direcciones)
├── Kubernetes (nodos + pods)         /22 por zona
│   ├── zona a   10.10.0.0/22
│   ├── zona b   10.10.4.0/22
│   └── zona c   10.10.8.0/22
├── Load Balancers                    /24 por zona
│   ├── zona a   10.10.12.0/24
│   ├── zona b   10.10.13.0/24
│   └── zona c   10.10.14.0/24
├── Bases de datos                    /26 por zona
│   ├── zona a   10.10.15.0/26
│   ├── zona b   10.10.15.64/26
│   └── zona c   10.10.15.128/26
└── Libre / crecimiento   10.10.15.192/26
```

### Cómo se materializa en cada nube

Cada proveedor modela la red de forma diferente. El mismo plan de /20 se aplica así:

**AWS — red regional, subnets por Availability Zone.**
La correspondencia es directa: cada `/22`, `/24` y `/26` es una subnet en una AZ concreta.
- VPC con CIDR `10.10.0.0/20`.
- 3 subnets privadas `/22` (nodos/pods EKS), 3 subnets públicas `/24` (LB/NLB), 3 subnets
  aisladas `/26` (RDS, sin internet gateway).
- Con VPC-CNI cada pod recibe una IP de la subnet; si se requiere mayor densidad de pods se
  usa un CIDR secundario en espacio CGNAT (`100.64.0.0/10`), sin tocar el /20 principal.

**Azure — red regional; la subnet abarca las zonas de la región.**
En Azure la zona es una propiedad del recurso, no de la subnet. El /20 del VNet se divide
**por capa** (no por zona):
- VNet `10.10.0.0/20`.
- `snet-aks` /22, `snet-lb` /24, `snet-db` /26 (con *delegation* a la base gestionada).
- Con **Azure CNI Overlay** los pods usan un rango overlay propio y no consumen IPs del VNet.

**GCP — el VPC es global; la subnet es regional (abarca zonas).**
La subnet cubre todas las zonas de su región, por lo que el /20 también se divide **por capa**:
- VPC global; subnet regional `10.10.0.0/22` para nodos.
- GKE usa **alias IP / secondary ranges** nativos: rango secundario `pods` y rango secundario
  `services`, definidos sobre la misma subnet.
- El LB interno emplea un *proxy-only subnet* `/24`; las bases gestionadas se conectan vía
  Private Service Access `/26`.

**Diferencia clave a retener:** en AWS la subnet vive en **una sola AZ** (de ahí la partición
por zona); en Azure y GCP la subnet **abarca varias zonas** de la región, por lo que el mismo
/20 se divide por capa de servicio. Además, en GCP el VPC es **global** y admite recursos en
varias regiones sin peering.

---

## 3. Dominios y DNS centralizados

**Principio:** el **registro** del dominio y la **delegación** se gestionan en un único lugar
(una cuenta o proyecto dedicado a DNS). Los dominios no se registran en cuentas personales.

| Nube      | Servicio DNS gestionado |
|-----------|-------------------------|
| AWS       | Route 53 |
| Azure     | Azure DNS |
| GCP       | Cloud DNS |
| Multi/CDN | Cloudflare (recomendado como capa central con WAF/CDN) |

### Reglas para los registros
- **Evitar registros `A` con IP fija manual.** Una IP fija queda desactualizada cuando el
  recurso rota, provocando indisponibilidad.
- Preferir **`CNAME`** apuntando al hostname del recurso (el LB o CDN gestiona las IPs).
- En **AWS**, para el ápex del dominio (que por RFC no admite CNAME) usar **Alias records** de
  Route 53 hacia ELB / CloudFront / S3 (sin costo por consulta).
- El equivalente al ápex en **Azure DNS** son los **Alias records**; en **GCP/Cloudflare** se
  resuelve con **CNAME flattening**.

### Delegación
- Los subdominios por entorno (`staging.example.com`, `dev.example.com`) se delegan con
  registros **`NS`** hacia zonas separadas, de modo que producción no dependa de cambios en
  entornos inferiores.

---

## 4. Convención de nombres (naming)

**Principio:** el nombre debe indicar **qué es y dónde está** sin necesidad de abrir la consola.

```
<environment>-<servicio>-<region>-<tipo-de-recurso>
```

Ejemplos:
```
prod-frontend-us-east-1-eks
prod-api-us-east-1-rds
staging-api-eastus-aks
dev-worker-southamerica-east1-gke
prod-network-us-east-1-vpc
```

| Segmento     | Valores de ejemplo                                    |
|--------------|-------------------------------------------------------|
| environment  | `prod` · `staging` · `dev`                            |
| servicio     | `frontend` · `api` · `worker` · `network`            |
| region       | `us-east-1` (AWS) · `eastus` (Azure) · `us-east1` (GCP) |
| tipo-recurso | `eks`/`aks`/`gke` · `vpc` · `rds` · `lb` · `bucket`  |

- **Minúsculas, separado por guiones.** Sin mayúsculas, espacios ni guiones bajos.
- El naming se complementa con **tags/labels** obligatorios (ver sección 10): el nombre es
  para lectura humana; los tags habilitan facturación y automatización.

---

## 5. SSO / Identidad

**Principio:** el acceso a las nubes y a las aplicaciones SaaS se realiza mediante **SSO**
desde un único **Identity Provider (IdP)**. No se crean credenciales locales por servicio.

| IdP                    | Contexto adecuado |
|------------------------|-------------------|
| **Okta**               | Amplio ecosistema de SaaS de terceros |
| **Google Workspace**   | Organizaciones ya basadas en Google Workspace |
| **Microsoft Entra ID** | Entornos centrados en Azure / Microsoft 365 |

- El IdP se federa a las tres nubes vía **SAML/OIDC** (AWS IAM Identity Center, Entra ID para
  Azure, Workforce Identity para GCP).
- **MFA obligatorio** para todas las cuentas.
- Las **identidades de máquina** (CI/CD, workloads) no usan SSO: emplean **roles federados por
  OIDC**, evitando credenciales estáticas de larga vida.
- El **alta y baja de personas** se gestiona en el IdP: desactivar a un usuario revoca su
  acceso a todos los sistemas de forma simultánea.

---

## 6. Control de código — repositorios, aprobaciones y CODEOWNERS

**Principio:** ningún cambio llega a la rama principal sin revisión. El historial de Git es la
traza de auditoría.

- Estandarizar sobre **GitHub** o **GitLab** (uno de los dos).
- **Protección de la rama `main`:**
  - Sin push directo.
  - Pull/Merge Request obligatorio con **1–2 aprobaciones** mínimas.
  - Checks de CI en verde (lint, tests, build) requeridos para hacer merge.
  - Firmado de commits (recomendado).
- **CODEOWNERS:** asigna revisores automáticos por ruta.
  ```
  # .github/CODEOWNERS  (o CODEOWNERS en GitLab)
  /app/        @org/backend
  /web/        @org/frontend
  /infra/      @org/platform
  *.tf         @org/platform
  ```
- **Nombres de repositorio:** `<org>-<proyecto>[-<componente>]`, en minúsculas y con guiones.
- **Nombres de rama:** `feat/…`, `fix/…`, `chore/…`, `hotfix/…`.
- La **infraestructura como código** vive en su propio repositorio, con el `plan` visible en el
  PR antes de aplicar.

---

## 7. Gestor de contraseñas corporativo

**Principio:** las credenciales no se comparten por mensajería, planillas ni archivos `.env`.

- Gestor de contraseñas de equipo: **1Password**, **Bitwarden** o **Vaultwarden** (self-hosted).
- **Vaults por equipo o proyecto** con acceso segmentado.
- El gestor de contraseñas guarda **secretos de personas** (logins de SaaS, tarjetas). Los
  **secretos de máquina** (API keys, contraseñas de base de datos) se almacenan en el
  **secrets manager de la nube** (AWS Secrets Manager / Azure Key Vault / GCP Secret Manager),
  nunca en el gestor de contraseñas.
- Idealmente, el propio gestor de contraseñas se accede mediante **SSO** (sección 5).

---

## 8. Sistema de tickets

**Principio:** todo pedido, incidente o cambio queda registrado. Lo que no está en un ticket,
no ocurrió.

- **Gestión interna:** Jira, Linear, o GitHub/GitLab Issues para lo técnico.
- **Soporte externo:** un helpdesk que convierta email en ticket (Zammad, Freshdesk u opción
  equivalente).
- **Gestión de cambios:** cada cambio en producción se documenta en un ticket con responsable,
  descripción, fecha y plan de rollback, vinculado al PR de Git.
- **Prefijos por proyecto** para trazabilidad (por ejemplo `INFRA-`, `SEC-`, `OPS-`).

---

## 9. Cómo encajan las piezas

```
        IdP (Okta / Google / Entra) ──SSO/MFA──┐
                                                ▼
   ┌────────── Root / Org / Tenant  (billing + guardrails) ───────────┐
   │  DNS Hub (Route53/Cloudflare)   Secrets Manager   Log Archive     │
   └──────────────────────────────────────────────────────────────────┘
        │  proj×env = account / subscription / project,  /20 CIDR c/u
        ▼
   prod-<project>    staging-<project>    dev-<project>    ...
        │
   Git (PR + CODEOWNERS) → CI/CD (OIDC, sin llaves) → IaC → recursos con naming + tags
        │
   Sistema de tickets (todo cambio queda registrado)
```

---

## 10. Componentes complementarios de la baseline

Elementos que suelen omitirse en la definición inicial y que conviene incorporar desde el
principio:

1. **Estrategia de tags/labels obligatorios** — `env`, `project`, `owner`, `cost-center`,
   `managed-by`. Sin ellos no es posible distribuir la facturación ni automatizar.
2. **Infraestructura como código** — Terraform/OpenTofu o Pulumi. Los cambios en producción no
   se realizan por consola; la consola queda como acceso de lectura.
3. **Logging y auditoría centralizados** — CloudTrail / Azure Activity Log / Cloud Audit Logs
   enviados a una cuenta de **log-archive inmutable**, no modificable ni siquiera por
   administradores.
4. **Backup y Disaster Recovery** — política 3-2-1, copias cross-account/region y **RTO/RPO**
   definidos por escrito. Un backup no probado mediante restauración no cuenta como backup.
5. **Presupuestos y alertas de costo** — AWS Budgets / Azure Cost Alerts / GCP Budgets con
   avisos en umbrales (50/80/100 %).
6. **Cuenta break-glass / de emergencia** — una identidad de acceso de último recurso con MFA
   físico, para el caso de indisponibilidad del IdP. Documentada y auditada en cada uso.
7. **Baseline de seguridad** — detección de amenazas (GuardDuty / Defender / Security Command
   Center), cifrado at-rest por defecto, almacenamiento privado por default y escaneo de
   vulnerabilidades en el pipeline.
8. **Gestión de dispositivos (MDM)** — equipos con disco cifrado y bloqueo automático.
9. **Runbook de respuesta a incidentes** — responsables, canales de comunicación y método de
   documentación definidos por adelantado.
10. **Registro de proveedores / SaaS** — inventario de herramientas con responsable, costo y
    fecha de renovación, para controlar la proliferación de SaaS.
11. **Clasificación de datos y cumplimiento** — identificación de datos sensibles (PII, datos
    de tarjetas → PCI DSS) y de su ubicación.
12. **Entorno sandbox** — cuenta descartable para experimentación, con límite de presupuesto
    agresivo y borrado automático.

---

*Documento de referencia — versión 1. Publicado por [Delzo.cloud](https://delzo.cloud) bajo
licencia [CC BY 4.0](../LICENSE). Recordá: es una guía de ejemplo, adaptala a tu infraestructura.*