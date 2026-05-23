# App Design Definitions

Repositorio central para documentación técnica y decisiones de arquitectura.  
Aquí se consolidan estándares de diseño, definiciones por aplicación/módulo, configuración de entornos y lineamientos de CI/CD.

## Estructura base

```text
.
├── architecture/
│   ├── frontend-web/
│   │   ├── react/
│   │   ├── astro/
│   │   └── tailwind/
│   └── mobile-games/
│       ├── csharp/
│       └── unity/
├── devops/
│   ├── environments/
│   │   ├── development/
│   │   ├── staging/
│   │   └── production/
│   └── cicd/
│       ├── github-actions/
│       └── sftp/
├── team-rules/
├── template-architecture.md
└── opencode.json
```

## Propósito de cada carpeta

- `architecture/frontend-web/`: definiciones para aplicaciones web modernas (React, Astro, Tailwind).
- `architecture/mobile-games/`: definiciones para aplicaciones móviles y juegos (C#, Unity).
- `devops/environments/`: configuración y diferencias por entorno (Desarrollo, Staging, Producción).
- `devops/cicd/`: flujos de automatización, pipelines y despliegues (GitHub Actions, SFTP).
- `team-rules/`: estándares transversales del equipo (naming, versionado, code review, release process).
- `template-architecture.md`: plantilla base para documentar nuevas aplicaciones o módulos.
- `opencode.json`: configuración de workspace para Copilot y plugin `superpowers`.

## Cómo contribuir

1. Crea o actualiza documentación en la carpeta correspondiente.
2. Usa `template-architecture.md` como base para nuevas definiciones.
3. Mantén trazabilidad de decisiones (fecha, contexto, impacto y responsables).
4. Abre un Pull Request con:
   - objetivo claro,
   - alcance delimitado,
   - cambios en estructura o estándares (si aplica).
5. Solicita revisión técnica antes de merge.

## Convención recomendada para nuevos documentos

- Nombre de archivo: `architecture-<dominio>-<modulo>.md`
- Ejemplos:
  - `architecture-web-checkout.md`
  - `architecture-mobile-auth.md`
  - `architecture-devops-release-pipeline.md`