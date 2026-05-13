# FacturaDirecta API Documentation Repository

Repositorio solo de documentación — sin código, sin sistema de build.
Auto-generado desde fuentes internas. La fuente no es pública.

## Qué es este repositorio

Referencia pública de la API de FacturaDirecta. Cada fichero Markdown
documenta un recurso o un tema transversal.

## Dónde vive la especificación

La especificación OpenAPI (contrato machine-readable) **no está en este
repositorio**. Se sirve viva en:

- `https://app.facturadirecta.com/openapi.json`

Esa URL es la fuente autoritativa. El Markdown de este repo es la capa
editorial que explica cómo usar la API.

## Estructura

- `sections/<resource>.md` — un fichero por recurso de la API.
- `guides/<topic>.md` — un fichero por tema transversal (auth,
  paginación, facturación electrónica, etc.).

## Formato

- Headers ATX (`#`, `##`, `###`...). No Setext.
- Frontmatter YAML mínimo: `title`, `audience`, `status`.
- El body empieza con un H1 que coincide con `title`.
- En páginas de recurso, secciones automáticas (`## Endpoints`,
  `## Scopes`) inyectadas por el sistema de docs aparecen al final o
  inline donde corresponda.

## Reglas duras

- **No inventar**. Documenta solo lo que está en
  `https://app.facturadirecta.com/openapi.json` o en el comportamiento real de la API.
- **No editar a mano**. Cualquier edición a este repo se sobreescribe
  en la siguiente publicación. La fuente es interna y no pública;
  cualquier propuesta de cambio se canaliza por feedback (ver README).
- **No añadir ficheros fuera de** `sections/`, `guides/`, `README.md`,
  `AGENTS.md`.

## Commits

Auto-generados por el publisher tras cada publicación. Cada commit
representa un snapshot completo de la documentación en ese punto.
