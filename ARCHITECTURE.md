# ARCHITECTURE.md — k8s-firecrawl-pocharlies

> Firecrawl (API de scraping web) autoalojado: API + servicio Playwright sobre RabbitMQ, Valkey y Postgres
> compartidos. Solo manifiestos. Escrito por `architect` (SC-1426).

## 1. Clientes y versiones

| cliente | repositorio / ruta | versión desplegada | cómo se despliega |
|---|---|---|---|
| `firecrawl-api` (:3002) + `firecrawl-playwright` (:3003) | `k8s/manifest.yaml` (un fichero) | imágenes por digest: `ghcr.io/firecrawl/firecrawl@sha256:11c96ce5…`, `ghcr.io/firecrawl/playwright-service@sha256:9e0737bc…` | ArgoCD app `firecrawl` |
| Publicación | IngressRoute `firecrawl-public` → `firecrawl.e-dani.com` (external-dns) | — | ídem |

## 2. Dependencias, en ambos sentidos

- **Depende de** — namespace `skirmshop` (todo vive ahí); RabbitMQ compartido (`rabbitmq-secrets`), Valkey
  `shared-valkey-master.databases` (secret `firecrawl-shared-valkey`), Postgres `postgres-shared-rw.databases`
  (bd `firecrawl`, secret `shared-postgres-app`), `labels-secrets`; initContainers `wait-for-shared-rabbitmq` y
  `wait-for-shared-postgres` (busybox 1.36). Rama local `it/infra-274-postgres-init` en curso sobre el init de BD.
- **Dependen de él** — scrapers del estate (crawler de competidores de Skirmshop, agentes con herramientas de
  scraping) que llaman a `https://firecrawl.e-dani.com` o `firecrawl-api.skirmshop.svc:3002`. Consumidores exactos:
  **pendiente** de `rg` en el resto de repos.
- **ArgoCD** `firecrawl`: repo `pocharlies-org/k8s-firecrawl-pocharlies`, path `k8s`, tronco **`main`**
  (`origin/main` = fb27b7b), sync automático, namespace `skirmshop`.

## 3. Stack

| pieza | versión | para qué | no se usa en su lugar |
|---|---|---|---|
| Firecrawl | digest (sin tag legible) | API de scraping | scrapers a medida |
| Playwright service de Firecrawl | digest | render JS | navegador propio |
| Kustomize (directorio plano) | — | render | Helm |

## 4. Componentes compartidos

| concepto | pieza canónica | ruta | quién la usa |
|---|---|---|---|
| Cola / caché / BD | RabbitMQ, Valkey, `postgres-shared` | `k8s-infra-pocharlies` (`databases`) | firecrawl y otros |
| CI estándar | `reusable-ci.yml@main` | `k8s-gitops-pocharlies` | `ci.yml` |

## 5. Cómo se construye aquí

Un único `manifest.yaml`. `ENV=local` limita los workers de la API a 2; `IS_KUBERNETES=true` usa límites de cgroup;
`USE_DB_AUTHENTICATION=false` (sin auth de usuario Firecrawl, la frontera es la red/ruta). Actualizar versión =
nuevo digest en ambas imágenes.

## 6. Tests y validaciones

Sin tests propios; `kustomize build . k8s` lo ejecuta el CI estándar.

## 7. CI/CD y despliegue

- `ci.yml` → `reusable-ci.yml@main` (`arc-k8s`, `kustomize_paths: ". k8s"`), `pr-review.yml`. En `origin/main` **no**
  existen `release.yml` ni `update-versions.yml` (están solo en la rama local de trabajo).
- Despliegue: merge a `main` → ArgoCD. **Validación en producción**: `POST https://firecrawl.e-dani.com/v1/scrape`
  con una URL simple y comprobar contenido; Synced ≠ funcionando. Pendiente de ejecutar.

## 8. Decisiones y trampas

- README desfasado («k3s v1.32.5», enlace a repo gitops en org `pocharlies`).
- Vive en el namespace `skirmshop` aunque es infraestructura transversal (acopla ciclo de vida con skirmshop).
- Imágenes pinneadas por digest sin tag: auditar versión real antes de subir (`upstream-autoupdate`).

Última verificación contra el código: 2026-10-01 · fb27b7b (origin/main)
