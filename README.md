# App Design Definitions

Repositorio central para guardar definiciones de diseño y decisiones de arquitectura de distintas aplicaciones y módulos.

## Estructura base

```text
.
├── applications/
│   └── <app-o-modulo>/
├── template-architecture.md
└── opencode.json
```

## Propósito de cada elemento

- `applications/`: archivador principal con una carpeta por aplicación o módulo.
- `template-architecture.md`: plantilla base para documentar una definición de diseño.
- `opencode.json`: configuración del workspace para mantener el enfoque del repositorio.

## Cómo usar este repositorio

1. Crea una carpeta dentro de `applications/` para la aplicación o módulo.
2. Usa `template-architecture.md` como base para el documento principal.
3. Mantén solo información útil al nivel de diseño: contexto, alcance, arquitectura, componentes, decisiones y riesgos.
4. Evita bajar a detalles de entornos, despliegues o automatizaciones si no afectan directamente el diseño.

## Convención recomendada

- Carpeta por aplicación o módulo: `applications/<app-o-modulo>/`
- Documento principal sugerido: `applications/<app-o-modulo>/architecture.md`
- Documentos adicionales opcionales:
  - `applications/<app-o-modulo>/decisions.md`
  - `applications/<app-o-modulo>/diagrams/`