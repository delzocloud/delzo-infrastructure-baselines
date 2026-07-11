# Infrastructure Baseline — Documento de Referência

**Idiomas:** [Español](../es/README.md) · [English](../en/README.md) · [Português](../pt/README.md)

> ### ⚠️ Aviso importante — leia antes de usar
>
> Estas baselines são um **exemplo de referência**, não uma configuração pronta para produção.
> Cada infraestrutura tem requisitos diferentes: escala, orçamento, conformidade regulatória,
> região geográfica e maturidade da equipe. Trate este material como um **guia** e
> **adapte-o** ao seu caso.
>
> Antes de implementar qualquer decisão descrita aqui, revise-a com um profissional que
> conheça o seu contexto. **A Delzo.cloud não se responsabiliza** por implementações baseadas
> neste documento sem a devida revisão.

---

> Conjunto de decisões de base que convém tomar **antes** de provisionar o primeiro recurso na
> nuvem. Definir isso cedo evita, mais adiante, problemas de faturamento difuso, sobreposição
> de faixas de rede, DNS gerenciado manualmente e falta de rastreabilidade sobre quem aprovou
> cada mudança.
>
> O documento se aplica a **AWS**, **Azure** e **GCP**. Como cada provedor modela a rede e a
> organização de contas de forma diferente, cada seção inclui os três.

---

## 1. Separação por conta / assinatura / projeto

**Princípio:** uma unidade de faturamento isolada por **projeto × environment**. Os ambientes
`prod` e `dev` nunca compartilham conta. Isso permite um faturamento limpo, isola o impacto de
um erro e habilita permissões independentes por ambiente.

| Nuvem | Unidade de isolamento              | Contêiner raiz                      |
|-------|------------------------------------|-------------------------------------|
| AWS   | **Account** (uma por proj×env)     | **Organization** com Management (Root) Account |
| Azure | **Subscription** (uma por proj×env) | **Management Groups** sob o Tenant |
| GCP   | **Project** (um por proj×env)      | **Organization** + **Folders**      |

### AWS
```
Management (Root) Account  ── apenas faturamento + Organizations, sem workloads
├── OU: Security       → account: log-archive, account: audit
├── OU: Infrastructure → account: network (hub), account: shared-services
└── OU: Workloads
    ├── OU: Prod       → account: <org>-<project>-prod
    ├── OU: Staging    → account: <org>-<project>-staging
    └── OU: Dev        → account: <org>-<project>-dev
```
- A Management Account **não hospeda recursos**: limita-se ao AWS Organizations, consolidated
  billing e **SCPs** (Service Control Policies).
- As **SCPs** por OU atuam como guardrails (por exemplo: restringir as regiões habilitadas ou
  impedir que o CloudTrail seja desativado).

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
- A **Azure Policy** aplicada sobre os Management Groups cumpre o papel de guardrail,
  equivalente às SCPs da AWS.

### GCP
```
Organization
├── Folder: prod     → project: <org>-<project>-prod
├── Folder: staging  → project: <org>-<project>-staging
├── Folder: dev      → project: <org>-<project>-dev
└── Folder: shared   → project: <org>-network-hub, project: <org>-logging
```
- **Organization Policies** por Folder atuam como guardrail.
- O faturamento é centralizado em uma **Billing Account** vinculada a todos os projects.

---

## 2. Endereçamento IP / CIDR

**Princípio:** atribuir um bloco **/20** (4.096 endereços) por **projeto × environment**, a
partir de um plano de endereçamento definido de antemão. As faixas não são deixadas ao default
do provedor: caso contrário, ao interconectar redes (peering / VPN) as faixas se sobrepõem e a
conexão se torna inviável.

### Por que /20 e não /16

Um **/16** oferece ~65.000 endereços. Na prática, um ambiente raramente consome uma fração
significativa disso, então reservar um /16 por ambiente **desperdiça espaço de endereçamento**
e esgota antes o plano global da organização.

Um **/20** (4.096 endereços) é suficiente e eficiente. O detalhamento a seguir mostra que um
único /20 acomoda um cluster de Kubernetes multi-zona, seus balanceadores e seus bancos de
dados, e ainda deixa endereços livres:

| Uso                         | Prefixo | Endereços | Comentário |
|-----------------------------|---------|-----------|------------|
| Nós + pods Kubernetes       | **/22 por zona** | ~1.024 por zona | Com 3 zonas: folgado para a maioria das cargas, ainda mais com Fargate/serverless |
| Load Balancers              | **/24 por zona** | 256 por zona | |
| Bancos de dados             | **/26 por zona** | 64 por zona | Isolados, sem rota para a internet |
| Reservado / crescimento     | restante | —        | Sobra espaço livre dentro do mesmo /20 |

> **Nota de capacidade:** ~1.000 endereços por zona para nós e pods cobre a enorme maioria dos
> cenários. Apenas uma aplicação de grande escala concentrada em um único cluster exigiria
> mais; nesse caso, atribui-se outro /20 a esse projeto, em vez de superdimensionar todos os
> ambientes com um /16.

### Exemplo de partição de um /20 (`10.10.0.0/20`)

```
10.10.0.0/20  (4.096 endereços)
├── Kubernetes (nós + pods)           /22 por zona
│   ├── zona a   10.10.0.0/22
│   ├── zona b   10.10.4.0/22
│   └── zona c   10.10.8.0/22
├── Load Balancers                    /24 por zona
│   ├── zona a   10.10.12.0/24
│   ├── zona b   10.10.13.0/24
│   └── zona c   10.10.14.0/24
├── Bancos de dados                   /26 por zona
│   ├── zona a   10.10.15.0/26
│   ├── zona b   10.10.15.64/26
│   └── zona c   10.10.15.128/26
└── Livre / crescimento   10.10.15.192/26
```

### Como se materializa em cada nuvem

Cada provedor modela a rede de forma diferente. O mesmo plano de /20 se aplica assim:

**AWS — rede regional, subnets por Availability Zone.**
A correspondência é direta: cada `/22`, `/24` e `/26` é uma subnet em uma AZ específica.
- VPC com CIDR `10.10.0.0/20`.
- 3 subnets privadas `/22` (nós/pods EKS), 3 subnets públicas `/24` (LB/NLB), 3 subnets
  isoladas `/26` (RDS, sem internet gateway).
- Com o VPC-CNI cada pod recebe um IP da subnet; se for necessária maior densidade de pods,
  usa-se um CIDR secundário em espaço CGNAT (`100.64.0.0/10`), sem tocar no /20 principal.

**Azure — rede regional; a subnet abrange as zonas da região.**
No Azure a zona é uma propriedade do recurso, não da subnet. O /20 da VNet é dividido **por
camada** (não por zona):
- VNet `10.10.0.0/20`.
- `snet-aks` /22, `snet-lb` /24, `snet-db` /26 (com *delegation* ao banco gerenciado).
- Com o **Azure CNI Overlay** os pods usam uma faixa overlay própria e não consomem IPs da VNet.

**GCP — a VPC é global; a subnet é regional (abrange zonas).**
A subnet cobre todas as zonas de sua região, portanto o /20 também é dividido **por camada**:
- VPC global; subnet regional `10.10.0.0/22` para nós.
- O GKE usa **alias IP / secondary ranges** nativos: faixa secundária `pods` e faixa
  secundária `services`, definidas sobre a mesma subnet.
- O LB interno usa uma *proxy-only subnet* `/24`; os bancos gerenciados se conectam via Private
  Service Access `/26`.

**Diferença-chave a reter:** na AWS a subnet vive em **uma única AZ** (daí a partição por
zona); no Azure e no GCP a subnet **abrange várias zonas** da região, portanto o mesmo /20 é
dividido por camada de serviço. Além disso, no GCP a VPC é **global** e admite recursos em
várias regiões sem peering.

---

## 3. Domínios e DNS centralizados

**Princípio:** o **registro** do domínio e a **delegação** são gerenciados em um único lugar
(uma conta ou projeto dedicado a DNS). Os domínios não são registrados em contas pessoais.

| Nuvem     | Serviço DNS gerenciado |
|-----------|------------------------|
| AWS       | Route 53 |
| Azure     | Azure DNS |
| GCP       | Cloud DNS |
| Multi/CDN | Cloudflare (recomendado como camada central com WAF/CDN) |

### Regras para os registros
- **Evitar registros `A` com IP fixo manual.** Um IP fixo fica desatualizado quando o recurso
  é rotacionado, provocando indisponibilidade.
- Preferir **`CNAME`** apontando para o hostname do recurso (o LB ou CDN gerencia os IPs).
- Na **AWS**, para o ápice do domínio (que por RFC não admite CNAME) usar **Alias records** do
  Route 53 para ELB / CloudFront / S3 (sem custo por consulta).
- O equivalente ao ápice no **Azure DNS** são os **Alias records**; no **GCP/Cloudflare** se
  resolve com **CNAME flattening**.

### Delegação
- Os subdomínios por ambiente (`staging.example.com`, `dev.example.com`) são delegados com
  registros **`NS`** para zonas separadas, de modo que a produção não dependa de mudanças em
  ambientes inferiores.

---

## 4. Convenção de nomes (naming)

**Princípio:** o nome deve indicar **o que é e onde está** sem necessidade de abrir o console.

```
<environment>-<serviço>-<region>-<tipo-de-recurso>
```

Exemplos:
```
prod-frontend-us-east-1-eks
prod-api-us-east-1-rds
staging-api-eastus-aks
dev-worker-southamerica-east1-gke
prod-network-us-east-1-vpc
```

| Segmento      | Valores de exemplo                                    |
|---------------|-------------------------------------------------------|
| environment   | `prod` · `staging` · `dev`                            |
| serviço       | `frontend` · `api` · `worker` · `network`            |
| region        | `us-east-1` (AWS) · `eastus` (Azure) · `us-east1` (GCP) |
| tipo-recurso  | `eks`/`aks`/`gke` · `vpc` · `rds` · `lb` · `bucket`  |

- **Minúsculas, separado por hífens.** Sem maiúsculas, espaços ou underscores.
- O naming é complementado por **tags/labels** obrigatórias (ver seção 10): o nome é para
  leitura humana; as tags habilitam faturamento e automação.

---

## 5. SSO / Identidade

**Princípio:** o acesso às nuvens e às aplicações SaaS é feito via **SSO** a partir de um único
**Identity Provider (IdP)**. Não se criam credenciais locais por serviço.

| IdP                    | Contexto adequado |
|------------------------|-------------------|
| **Okta**               | Amplo ecossistema de SaaS de terceiros |
| **Google Workspace**   | Organizações já baseadas no Google Workspace |
| **Microsoft Entra ID** | Ambientes centrados em Azure / Microsoft 365 |

- O IdP é federado às três nuvens via **SAML/OIDC** (AWS IAM Identity Center, Entra ID para
  Azure, Workforce Identity para GCP).
- **MFA obrigatório** para todas as contas.
- As **identidades de máquina** (CI/CD, workloads) não usam SSO: empregam **roles federadas por
  OIDC**, evitando credenciais estáticas de longa duração.
- A **entrada e saída de pessoas** é gerenciada no IdP: desativar um usuário revoga seu acesso
  a todos os sistemas de forma simultânea.

---

## 6. Controle de código — repositórios, aprovações e CODEOWNERS

**Princípio:** nenhuma mudança chega à branch principal sem revisão. O histórico do Git é a
trilha de auditoria.

- Padronizar em **GitHub** ou **GitLab** (um dos dois).
- **Proteção da branch `main`:**
  - Sem push direto.
  - Pull/Merge Request obrigatório com **1–2 aprovações** mínimas.
  - Checks de CI em verde (lint, tests, build) obrigatórios para fazer merge.
  - Assinatura de commits (recomendado).
- **CODEOWNERS:** atribui revisores automáticos por caminho.
  ```
  # .github/CODEOWNERS  (ou CODEOWNERS no GitLab)
  /app/        @org/backend
  /web/        @org/frontend
  /infra/      @org/platform
  *.tf         @org/platform
  ```
- **Nomes de repositório:** `<org>-<projeto>[-<componente>]`, em minúsculas e com hífens.
- **Nomes de branch:** `feat/…`, `fix/…`, `chore/…`, `hotfix/…`.
- A **infraestrutura como código** vive em seu próprio repositório, com o `plan` visível no PR
  antes de aplicar.

---

## 7. Gerenciador de senhas corporativo

**Princípio:** as credenciais não são compartilhadas por mensageria, planilhas ou arquivos
`.env`.

- Gerenciador de senhas de equipe: **1Password**, **Bitwarden** ou **Vaultwarden**
  (self-hosted).
- **Vaults por equipe ou projeto** com acesso segmentado.
- O gerenciador de senhas guarda **segredos de pessoas** (logins de SaaS, cartões). Os
  **segredos de máquina** (API keys, senhas de banco de dados) são armazenados no **secrets
  manager da nuvem** (AWS Secrets Manager / Azure Key Vault / GCP Secret Manager), nunca no
  gerenciador de senhas.
- Idealmente, o próprio gerenciador de senhas é acessado via **SSO** (seção 5).

---

## 8. Sistema de tickets

**Princípio:** todo pedido, incidente ou mudança fica registrado. O que não está em um ticket
não aconteceu.

- **Gestão interna:** Jira, Linear, ou GitHub/GitLab Issues para o que é técnico.
- **Suporte externo:** um helpdesk que converta e-mail em ticket (Zammad, Freshdesk ou opção
  equivalente).
- **Gestão de mudanças:** cada mudança em produção é documentada em um ticket com responsável,
  descrição, data e plano de rollback, vinculado ao PR do Git.
- **Prefixos por projeto** para rastreabilidade (por exemplo `INFRA-`, `SEC-`, `OPS-`).

---

## 9. Como as peças se encaixam

```
        IdP (Okta / Google / Entra) ──SSO/MFA──┐
                                                ▼
   ┌────────── Root / Org / Tenant  (billing + guardrails) ───────────┐
   │  DNS Hub (Route53/Cloudflare)   Secrets Manager   Log Archive     │
   └──────────────────────────────────────────────────────────────────┘
        │  proj×env = account / subscription / project,  /20 CIDR cada
        ▼
   prod-<project>    staging-<project>    dev-<project>    ...
        │
   Git (PR + CODEOWNERS) → CI/CD (OIDC, sem chaves) → IaC → recursos com naming + tags
        │
   Sistema de tickets (toda mudança fica registrada)
```

---

## 10. Componentes complementares da baseline

Elementos que costumam ser omitidos na definição inicial e que convém incorporar desde o
princípio:

1. **Estratégia de tags/labels obrigatórias** — `env`, `project`, `owner`, `cost-center`,
   `managed-by`. Sem elas não é possível distribuir o faturamento nem automatizar.
2. **Infraestrutura como código** — Terraform/OpenTofu ou Pulumi. As mudanças em produção não
   são feitas pelo console; o console fica como acesso somente de leitura.
3. **Logging e auditoria centralizados** — CloudTrail / Azure Activity Log / Cloud Audit Logs
   enviados a uma conta de **log-archive imutável**, não modificável nem mesmo por
   administradores.
4. **Backup e Disaster Recovery** — política 3-2-1, cópias cross-account/region e **RTO/RPO**
   definidos por escrito. Um backup não testado por meio de restauração não conta como backup.
5. **Orçamentos e alertas de custo** — AWS Budgets / Azure Cost Alerts / GCP Budgets com avisos
   em limiares (50/80/100 %).
6. **Conta break-glass / de emergência** — uma identidade de acesso de último recurso com MFA
   físico, para o caso de indisponibilidade do IdP. Documentada e auditada em cada uso.
7. **Baseline de segurança** — detecção de ameaças (GuardDuty / Defender / Security Command
   Center), criptografia at-rest por padrão, armazenamento privado por padrão e varredura de
   vulnerabilidades no pipeline.
8. **Gestão de dispositivos (MDM)** — equipamentos com disco criptografado e bloqueio
   automático.
9. **Runbook de resposta a incidentes** — responsáveis, canais de comunicação e método de
   documentação definidos com antecedência.
10. **Registro de fornecedores / SaaS** — inventário de ferramentas com responsável, custo e
    data de renovação, para controlar a proliferação de SaaS.
11. **Classificação de dados e conformidade** — identificação de dados sensíveis (PII, dados de
    cartões → PCI DSS) e de sua localização.
12. **Ambiente sandbox** — conta descartável para experimentação, com limite de orçamento
    agressivo e exclusão automática.

---

*Documento de referência — versão **1.0.0** ([histórico de versões](../CHANGELOG.md)).
Publicado por [Delzo.cloud](https://delzo.cloud) sob a licença [CC BY 4.0](../LICENSE).
Lembre-se: é um guia de exemplo, adapte-o à sua infraestrutura.*