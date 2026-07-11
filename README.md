# Delzo Infrastructure Baselines

Baselines de infraestructura cloud (AWS · Azure · GCP): las decisiones de base que conviene
tomar **antes** de aprovisionar el primer recurso — separación de cuentas, plan de
direccionamiento IP, DNS, naming, SSO, control de código y más.

Cloud infrastructure baselines (AWS · Azure · GCP): the foundational decisions worth making
**before** provisioning the first resource — account separation, IP addressing plan, DNS,
naming, SSO, code control and more.

Baselines de infraestrutura cloud (AWS · Azure · GCP): as decisões de base que convém tomar
**antes** de provisionar o primeiro recurso — separação de contas, plano de endereçamento IP,
DNS, naming, SSO, controle de código e mais.

---

## 📖 Leer / Read / Ler

| Idioma / Language | Documento |
|---|---|
| 🇪🇸 Español    | **[es/README.md](es/README.md)** |
| 🇬🇧 English    | **[en/README.md](en/README.md)** |
| 🇧🇷 Português  | **[pt/README.md](pt/README.md)** |

El contenido es idéntico en los tres idiomas · The content is identical across all three
languages · O conteúdo é idêntico nos três idiomas.

---

## ⚠️ Disclaimer

**ES —** Estas baselines son un **ejemplo de referencia**, no una configuración lista para
producción. Cada infraestructura tiene requisitos distintos (escala, presupuesto, cumplimiento
normativo, región). Tomá este material como una **guía** y **adaptalo** a tu caso. Delzo.cloud
no se responsabiliza por implementaciones basadas en este documento sin la debida revisión
profesional.

**EN —** These baselines are a **reference example**, not a production-ready configuration.
Every infrastructure has different requirements (scale, budget, regulatory compliance, region).
Treat this material as a **guide** and **adapt it** to your case. Delzo.cloud takes no
responsibility for implementations based on this document without proper professional review.

**PT —** Estas baselines são um **exemplo de referência**, não uma configuração pronta para
produção. Cada infraestrutura tem requisitos diferentes (escala, orçamento, conformidade
regulatória, região). Trate este material como um **guia** e **adapte-o** ao seu caso. A
Delzo.cloud não se responsabiliza por implementações baseadas neste documento sem a devida
revisão profissional.

---

## 🔖 Versionado / Versioning / Versionamento

Las baselines se versionan con [Semantic Versioning](https://semver.org/). Cada versión queda
registrada como un [GitHub Release](https://github.com/delzocloud/delzo-infrastructure-baselines/releases)
con su tag (`v1.0.0`, `v1.1.0`, …), y el detalle de cambios se lleva en el
**[CHANGELOG.md](CHANGELOG.md)**.

| Cambio | Ejemplo | Cuándo |
|---|---|---|
| **MAJOR** | `2.0.0` | Se cambia o revierte una recomendación existente (guía "breaking"). |
| **MINOR** | `1.1.0` | Se agrega una sección o recomendación nueva. |
| **PATCH** | `1.0.1` | Correcciones de redacción, ejemplos o traducciones. |

> Para citar una versión estable y reproducible, referenciá un tag concreto (por ejemplo
> `.../blob/v1.0.0/es/README.md`) en lugar de la rama `main`.

---

## 📄 Licencia / License / Licença

[CC BY 4.0](LICENSE) — podés compartir y adaptar el material con atribución · you may share and
adapt the material with attribution · você pode compartilhar e adaptar o material com
atribuição.

## 🌐 Delzo.cloud

Hecho por **[Delzo.cloud](https://delzo.cloud)** — *"Vendemos resultados, no tecnología."*
Consultoría de infraestructura cloud y DevOps para PyMEs de Latinoamérica.