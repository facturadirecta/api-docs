---
title: Configuración de la empresa
audience:
  - developers
status: draft
---

# Configuración de la empresa

Bajo `/settings/...` viven los catálogos de configuración de la empresa
que la API pública expone como **solo lectura**: impuestos disponibles,
series de numeración por tipo de documento y plantillas de impresión.
La modificación de esta configuración se hace desde la interfaz, no
por API.

## Las 4 operaciones

| Endpoint | Cubre |
|---|---|
| `GET /settings/taxes/sales` | Catálogo de impuestos para documentos de venta. Documentado en la [guía de Impuestos](../guides/taxes.md). |
| `GET /settings/taxes/purchases` | Catálogo de impuestos para documentos de compra. Documentado en la [guía de Impuestos](../guides/taxes.md). |
| `GET /settings/series/{documentType}` | Series de numeración configuradas en la empresa para un tipo de documento. |
| `GET /settings/themes` | Plantillas de impresión disponibles. |

Las cuatro requieren scope **`settings:read`**.

> Para los catálogos de impuestos consulta directamente la
> [guía de Impuestos](../guides/taxes.md) — cubre el modelo completo
> (dos catálogos, schema, autorepercutidos, `lineAmountIsTax`...).
> Esta página solo documenta las dos operaciones restantes (`series`
> y `themes`).

## Operaciones

- [Lista de series de numeración](#lista-de-series-de-numeración)
- [Lista de plantillas](#lista-de-plantillas)

## Lista de series de numeración

`GET /{companyId}/settings/series/{documentType}` devuelve las series
de numeración disponibles para un tipo de documento.

**Parámetro de path:**

- **`documentType`** (obligatorio) — uno de:
  - `invoice` — facturas de venta.
  - `estimate` — presupuestos.
  - `deliveryNote` — albaranes.

**Parámetros globales aceptados:** `accept-version`.

**Respuesta:** `{ items: SeriesItem[] }` sin paginar.

Cada `SeriesItem` tiene:

| Campo | Tipo | Significado |
|---|---|---|
| `serie` | string | Valor de la serie. Puede contener `##` o `####`, que se sustituye por el año de la fecha del documento en el momento de guardar. |
| `reset` | `"never"` \| `"year"` | Si reinicia la numeración con el cambio de año (`year`) o si continúa desde el último número del año anterior (`never`). |
| `manual` | boolean | Si la serie acepta numeración manual desde la interfaz. |
| `notes` | string | Notas descriptivas. |
| `theme` | string | ID de plantilla por defecto que se aplica a documentos de esta serie. |
| `invoiceType` | enum | Solo en series de facturas. Valores: `complete`, `complete_correction`, `simplified`, `simplified_correction`, `external`. |
| `correction` | boolean | **OBSOLETO** — usar `invoiceType` en su lugar. Indica si la serie es para rectificativas. |

**Notas:**

- Las series se configuran desde la interfaz; este endpoint solo las
  expone. La API pública no crea, modifica ni borra series.
- Para series de facturas, `invoiceType` distingue los regímenes
  fiscales aplicables. El campo `correction` queda por
  compatibilidad pero **no debe usarse en código nuevo**.
- `##`/`####` en `serie` se sustituye por el año al guardar el
  documento. Ejemplo: serie `"F-##-"` con fecha 2026 produce números
  como `F-26-1`, `F-26-2`...

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/settings/series/invoice"
```

## Lista de plantillas

`GET /{companyId}/settings/themes` devuelve las plantillas de
impresión configuradas en la empresa.

**Parámetros globales aceptados:** `accept-version`.

**Respuesta:** `{ items: ThemeItem[] }` sin paginar.

Cada `ThemeItem` tiene:

| Campo | Tipo | Significado |
|---|---|---|
| `id` | string | Identificador de la plantilla. Es lo que se usa en `content.main.theme` de los documentos. |
| `title` | string | Nombre legible. |

**Notas:**

- Las plantillas se configuran desde la interfaz; este endpoint solo
  las expone.
- Para aplicar una plantilla a un documento, indica su `id` en
  `content.main.theme` al crear o actualizar el documento.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/settings/themes"
```

## Recomendaciones

- **Cachea el catálogo** localmente por empresa. Series, themes e
  impuestos cambian con muy baja frecuencia; consultarlos en cada
  llamada gasta cuota innecesariamente.
- **Refresca al fallar**: si una creación falla con `400` por
  `theme` o `serie` desconocido, refresca el catálogo: alguien puede
  haber cambiado la configuración desde la interfaz.

## Referencia exhaustiva

Esta página cubre los matices funcionales de `series` y `themes`.
Para impuestos, la referencia funcional está en la
[guía de Impuestos](../guides/taxes.md). Para el contrato completo
campo a campo, consulta el
[Swagger UI](https://www.facturadirecta.com/api) o el
[openapi crudo](https://app.facturadirecta.com/openapi.json).

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| GET | `/{companyId}/settings/series/{documentType}` | `getSeries` | `settings:read` | Lista de series de numeración |
| GET | `/{companyId}/settings/taxes/purchases` | `getPurchasesTaxes` | `settings:read` | Lista de impuestos de compra disponibles |
| GET | `/{companyId}/settings/taxes/sales` | `getSalesTaxes` | `settings:read` | Lista de impuestos de venta disponibles |
| GET | `/{companyId}/settings/themes` | `getThemes` | `settings:read` | Lista de plantillas de documentos disponibles |

## Scopes

- **`settings:read`** — Lectura de ajustes/datos de configuración.
