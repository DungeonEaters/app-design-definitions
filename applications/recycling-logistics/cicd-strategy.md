# Estrategia de CI/CD — Logística y Recolección de Plásticos (MVP)

Define cómo se construyen, versionan, publican y despliegan las imágenes del proyecto tras el paso a polyrepo. Complementa [architecture.md](architecture.md) (ADR-013/014/015) y [decisions.md](decisions.md).

## 1. Repositorios y responsabilidad

| Repositorio | Responsabilidad en CI/CD |
|---|---|
| [recycling-backend](https://github.com/DungeonEaters/recycling-backend) | Test (pytest), build y publicación de imagen `api` en GHCR |
| [recycling-web](https://github.com/DungeonEaters/recycling-web) | Lint/build (npm), publicación de imagen `web` (Nginx) en GHCR |
| [recycling-mobile](https://github.com/DungeonEaters/recycling-mobile) | Test + build (npm), publicación de imagen `mobile` (Nginx) en GHCR |
| [recycling-infra](https://github.com/DungeonEaters/recycling-infra) | Terraform (provisión Oracle) + workflow de despliegue (pull de imágenes ya publicadas y `docker compose up`) |

Cada repo de aplicación es dueño de su propio ciclo de versiones; no existe un tag o release único de plataforma.

## 2. Versionado: SemVer en dos canales

Ambos canales usan [SemVer 2.0](https://semver.org/). La diferencia es el trigger y si llevan sufijo de pre-release.

| Canal | Trigger | Formato de versión | Tag de imagen GHCR | Tag de Git | GitHub Release |
|---|---|---|---|---|---|
| **test** | Manual (`workflow_dispatch`) | `vX.Y.Z-rc.N` / `-beta.N` / `-alpha.N` | `test-vX.Y.Z-rc.N` | No obligatorio (se puede construir desde cualquier rama/commit) | No |
| **release** | Automático al hacer `git push --tags` de `vX.Y.Z` | `vX.Y.Z` (sin sufijo) | `release-vX.Y.Z` + `latest` | Sí, es el trigger | Sí, se crea/confirma automáticamente con notas generadas |

En ambos casos se publica además un tag inmutable `sha-<commit>` para trazabilidad exacta.

**Por qué este esquema:** separa "quiero probar una build específica en el ambiente de test" (rápido, manual, sin ceremonia de Git) de "esto es una versión oficial" (disparada solo por un tag real, trazable en la pestaña Releases de GitHub). Al no auto-publicar en cada push a `main`, se evita construir/publicar imágenes de commits que aún no se consideran listos para probarse.

## 3. Flujo completo

```
PR  ──► ci.yml (lint/test/build, sin publicar) ──► merge a main
                                                         │
                         ┌───────────────────────────────┴───────────────────────────────┐
                         │                                                               │
        workflow_dispatch "release.yml" (canal test)                      git tag vX.Y.Z && git push --tags
        input: version=vX.Y.Z-rc.N, ref=rama/commit                                      │
                         │                                                 release.yml (canal release, automático)
                build + push GHCR:                                                        │
                test-vX.Y.Z-rc.N, sha-<commit>                               build + push GHCR:
                                                                          release-vX.Y.Z, latest, sha-<commit>
                                                                                           │
                                                                          crear/confirmar GitHub Release vX.Y.Z
                         │                                                               │
                         └───────────────────────────────┬───────────────────────────────┘
                                                          ▼
                                 recycling-infra: deploy.yml (workflow_dispatch manual)
                                 input: environment=test|production, component=all|api|web|mobile,
                                        image_tag=test-vX.Y.Z-rc.N | release-vX.Y.Z
                                                          ▼
                                 SSH a la VM Oracle → docker compose -f docker-compose.prod.yml pull/up
```

El despliegue **siempre** es manual (`workflow_dispatch` en `recycling-infra`) y recibe un tag de imagen ya publicado — nunca construye ni redeploya automáticamente al publicar una imagen. Esto da control explícito sobre qué versión corre en cada ambiente.

## 4. Workflows por repo de aplicación

Cada uno de `recycling-backend`, `recycling-web`, `recycling-mobile` implementa el mismo patrón:

- **`.github/workflows/ci.yml`**: en `pull_request` y `push` a `main`. Jobs: test/lint (pytest o `npm test`/`npm run build` según el repo). No publica nada; es el status check requerido para mergear a `main`.
- **`.github/workflows/release.yml`**:
  - Job común: valida formato SemVer de `version` (regex), build multi-stage ARM64 (`linux/arm64`, QEMU + Buildx), login a GHCR con `GITHUB_TOKEN` (`permissions: packages: write`).
  - Trigger `workflow_dispatch` (canal test): inputs `version` (SemVer con sufijo pre-release obligatorio) y `ref` (rama o commit a construir, default `main`). Publica `test-<version>` y `sha-<commit>`.
  - Trigger `push.tags: ["v*.*.*"]` (canal release): valida que el tag no tenga sufijo de pre-release, publica `release-<version>`, `latest` y `sha-<commit>`, y crea el GitHub Release (notas autogeneradas con `generate_release_notes: true`).

Imagen resultante: `ghcr.io/dungeoneaters/recycling-backend`, `ghcr.io/dungeoneaters/recycling-web`, `ghcr.io/dungeoneaters/recycling-mobile` (ya reflejado en `recycling-infra/.env.production.example`).

## 5. Branch protection y status checks

`main` en los 4 repos ya exige PR + 1 aprobación (ver ADR de protección de ramas en [decisions.md](decisions.md)). Una vez el job `ci.yml` exista y corra en PRs, se debe marcar ese job como **required status check** en la protección de rama de `recycling-backend`, `recycling-web` y `recycling-mobile` (no aplica aún a `recycling-infra`, que no tiene CI de aplicación, solo Terraform).

## 6. Relación con Terraform y despliegue a Oracle

- Los workflows `terraform-plan.yml`/`terraform-apply.yml` de `recycling-infra` provisionan la VM; no están acoplados al versionado de imágenes de aplicación.
- El workflow de despliegue (`deploy.yml`, ver sección de infraestructura en [oracle-cloud-provisioning.md](oracle-cloud-provisioning.md)) es el único punto donde un `image_tag` de GHCR llega a la VM. Usa `API_IMAGE`/`WEB_IMAGE`/`MOBILE_IMAGE` (definidos en `.env` de la VM, uno por repo) en vez de un único `IMAGE_REPOSITORY` como en el monorepo legado.
- Credenciales OCI necesarias para Terraform: ver [oracle-credentials-setup.md](oracle-credentials-setup.md).
