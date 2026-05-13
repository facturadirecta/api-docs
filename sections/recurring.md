---
title: Facturas de venta recurrentes
audience:
  - developers
status: draft
---

# Facturas de venta recurrentes

Una factura recurrente es una **plantilla de factura programada**: cuando
llega el momento configurado, FacturaDirecta genera automáticamente la
factura real a partir de la plantilla. Útil para suscripciones,
mantenimientos, alquileres, retainers y cualquier facturación periódica
con importes y conceptos estables.

> En los ejemplos de esta página:
>
> - Los **UUIDs** (`aut_…`, `con_…`, `upl_…`) son ilustrativos. Cada
>   empresa tiene los suyos; sustitúyelos por los identificadores reales
>   que devuelve la API.
> - El **ID de impuesto** `S_IVA_21` es el del catálogo por defecto
>   para IVA 21% en ventas. Recupera el catálogo real con
>   `GET /{companyId}/settings/taxes/sales`. Ver
>   [Impuestos](../guides/taxes.md).

## Conceptualmente: una automatización con `action: createDocument`

Una factura recurrente es **técnicamente una automatización**
(`content.type: "automation"`). Tiene un **trigger** (cuándo se dispara)
y una **action** (qué hace al dispararse, en este caso crear una factura).
El prefijo del identificador es `aut_<uuid>` (de "automation").

La API pública expone hoy un único uso de las automatizaciones: facturas
de venta recurrentes (`action.type: "createDocument"`,
`action.template.type: "invoice"`). El recurso puede extenderse a otras
recurrencias en el futuro.

## Doble scope

A diferencia del resto de recursos, cada operación de recurring requiere
**dos scopes simultáneamente**:

- Las operaciones de **lectura** requieren `recurring:read` **y**
  `invoices:read`.
- Las operaciones de **escritura** requieren `recurring:write` **y**
  `invoices:write`.

El motivo: la recurrente toca dos dominios — la programación (recurring)
y la factura que genera (invoices). Quien gestiona recurrentes debe
poder leer/escribir ambos.

## Estructura

- `content.type` — siempre `"automation"`.
- `content.uuid` — identificador inmutable, prefijo `aut_`.
- `content.main`:
  - `title` — nombre descriptivo de la recurrente.
  - `enabled` — `true`/`false`. Si `false`, las próximas ejecuciones
    programadas se omiten sin perderse en el calendario.
  - `trigger` — cuándo se dispara (ver [Trigger y rrule](#trigger-y-rrule)).
  - `action` — qué hace al dispararse (ver [Action: plantilla de factura](#action-plantilla-de-factura)).
- `content.attachments` — adjuntos vinculados (ver [Adjuntos](#adjuntos)).

### Trigger y `rrule`

El trigger sigue el estándar RFC 5545 (iCalendar) en una versión
simplificada. Forma:

```json
"trigger": {
  "type": "recurring",
  "rrule": {
    "freq": "monthly",
    "interval": 1,
    "dtstart": "2026-06-01T06:00:00.000Z",
    "until": "2027-06-01T06:00:00.000Z",
    "bymonthday": 1
  },
  "daysAdvance": 0
}
```

Campos de `rrule`:

- **`freq`** (obligatorio) — frecuencia base. Valores: `daily`, `weekly`,
  `monthly`, `yearly`.
- **`interval`** (obligatorio, entero ≥1) — cada cuántas unidades de
  `freq`. Ejemplos: `freq: "monthly", interval: 1` (mensual);
  `freq: "monthly", interval: 3` (trimestral); `freq: "weekly", interval: 2` (cada dos semanas).
- **`dtstart`** (obligatorio) — fecha-hora de inicio en ISO 8601.
  Solo se evalúan ejecuciones a partir de esta fecha.
- **`until`** o **`count`** (opcional, mutuamente excluyentes) — fin del
  calendario: por fecha o por número de ejecuciones totales.
- **`byweekday`** (opcional, 0-6) — para `freq` `weekly` o `monthly`,
  fija el día de la semana (0=lunes, 6=domingo).
- **`bymonthday`** (opcional) — para `freq` `monthly` o `yearly`, día
  del mes (1-31 o `-1` para el último día del mes). Para `freq: monthly`
  el máximo aceptado es 28 (para evitar meses sin ese día).
- **`bysetpos`** (opcional) — para `freq: monthly`, posición ordinal del
  día de la semana en el mes: 1-4 para "el primer/segundo/.../cuarto
  `byweekday`", `-1` para "el último `byweekday`". Requiere `byweekday`.
  Ejemplo: `byweekday: 2, bysetpos: -1` = "el último miércoles del mes".
- **`bymonth`** (opcional, 1-12) — para `freq: yearly`, mes del año.

Otros campos del trigger:

- **`daysAdvance`** — reservado para uso futuro. Indicar siempre `0`.

### Action: plantilla de factura

`content.main.action` describe qué documento se genera. Forma:

```json
"action": {
  "type": "createDocument",
  "template": {
    "type": "invoice",
    "main": {
      "docNumber": { "series": "FP" },
      "contact": "con_2c4f8e21-7b3a-4d9c-9e1f-a8b7c6d5e4f3",
      "currency": "EUR",
      "lines": [
        {
          "quantity": 1,
          "unitPrice": 100,
          "tax": ["S_IVA_21"],
          "text": "Mantenimiento mensual"
        }
      ]
    }
  }
}
```

`template.main` es una estructura equivalente a la de una factura de
venta normal (ver [Crear presupuesto](./estimates.md#crear-presupuesto)
para el patrón de campos en documentos de venta; `invoices.md` cuando
exista). Cuando llega el momento programado, FacturaDirecta crea una
factura con los datos de esta plantilla, asignando un nuevo número de
documento y la fecha actual.

## Operaciones

- [Lista de facturas recurrentes](#lista-de-facturas-recurrentes)
- [Crear factura recurrente](#crear-factura-recurrente)
- [Obtener una factura recurrente](#obtener-una-factura-recurrente)
- [Actualizar factura recurrente](#actualizar-factura-recurrente)
- [Borrar factura recurrente](#borrar-factura-recurrente)
- [Actualizar etiquetas](#actualizar-etiquetas)
- [Generar muestra en PDF](#generar-muestra-en-pdf)
- [Adjuntos](#adjuntos)

## Lista de facturas recurrentes

`GET /{companyId}/recurring/invoices` devuelve las facturas recurrentes
de la empresa, paginadas.

**Parámetros de consulta específicos:**

- **`minNextScheduledDate`**, **`maxNextScheduledDate`** — rango de
  fecha de la próxima ejecución programada (ISO 8601). Útil para
  identificar qué recurrentes están a punto de ejecutarse o cuáles ya
  vencieron.
- **`series`** — serie de numeración de la factura plantilla.
- **`enabled`** — `true`/`false` para filtrar activas/inactivas.
- **`freq`** — frecuencia (`daily`/`weekly`/`monthly`/`yearly`).
- **`interval`** — intervalo entero.
- **`contact`** — ID de contacto de la factura plantilla.
- **`hasContact`** — `true`/`false`.
- **`minTotal`**, **`maxTotal`** — rango de importe de la plantilla.
- **`currency`** — código ISO 4217.
- **`country`** — código país ISO 3166-1 Alpha-2.
- **`allTheseTags`**, **`anyOfTheseTags`**, **`hasTags`** — filtros por etiquetas.
- **`sortBy`** — campo de orden.
- **`related`** — valores admitidos:
  - `nextScheduledTime` — devuelve la próxima fecha-hora prevista.
  - `upcomingSceheduledTimes` — devuelve hasta las próximas 5 fechas
    programadas. (Nombre con typo histórico en el openapi; se mantiene
    por compatibilidad.)
  - `taxIds` — devuelve en `related.objects` el detalle de los impuestos
    aplicados en la plantilla.

**Parámetros globales aceptados:**

Acepta además los parámetros estándar `offset`, `limit`,
`minCreationDate`, `maxCreationDate`, `minModificationDate`,
`maxModificationDate` y el header `accept-version`. Ver
[Paginación](../guides/pagination.md) y
[Autenticación](../guides/authentication.md).

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/recurring/invoices?enabled=true&freq=monthly&related=nextScheduledTime"
```

## Crear factura recurrente

`POST /{companyId}/recurring/invoices` crea una factura recurrente.

**Parámetros del body:**

- `content.type` — siempre `"automation"`.
- `content.main.title` (obligatorio) — nombre descriptivo.
- `content.main.enabled` (opcional, por defecto `true`).
- `content.main.trigger` (obligatorio) — ver [Trigger y rrule](#trigger-y-rrule).
- `content.main.action` (obligatorio) — ver [Action: plantilla de factura](#action-plantilla-de-factura).
- `tags` (opcional).

**Notas:**

- La primera ejecución no se produce inmediatamente al crear: se produce
  cuando llega la primera fecha calculada según `trigger.rrule` a partir
  de `dtstart`.
- Si `enabled: false`, la recurrente se guarda pero las ejecuciones
  programadas se omiten hasta que se reactive.
- La respuesta es la recurrente creada completa, en el mismo formato que
  [Obtener una factura recurrente](#obtener-una-factura-recurrente).

**Parámetros globales aceptados:** `accept-version`.

###### Ejemplo de request JSON

Recurrente mensual el día 1 de cada mes, desde junio de 2026 durante
12 meses:

```json
{
  "content": {
    "type": "automation",
    "main": {
      "title": "Mantenimiento mensual - Acme",
      "enabled": true,
      "trigger": {
        "type": "recurring",
        "rrule": {
          "freq": "monthly",
          "interval": 1,
          "dtstart": "2026-06-01T06:00:00.000Z",
          "count": 12,
          "bymonthday": 1
        }
      },
      "action": {
        "type": "createDocument",
        "template": {
          "type": "invoice",
          "main": {
            "docNumber": { "series": "FP" },
            "contact": "con_2c4f8e21-7b3a-4d9c-9e1f-a8b7c6d5e4f3",
            "currency": "EUR",
            "lines": [
              {
                "quantity": 1,
                "unitPrice": 100,
                "tax": ["S_IVA_21"],
                "text": "Mantenimiento mensual"
              }
            ]
          }
        }
      }
    }
  }
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '@recurringInvoice.json' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/recurring/invoices"
```

## Obtener una factura recurrente

`GET /{companyId}/recurring/invoices/{id}` devuelve una factura
recurrente por ID.

**Parámetros específicos:**

- `related` (opcional) — mismos valores que en
  [Lista de facturas recurrentes](#lista-de-facturas-recurrentes)
  (`nextScheduledTime`, `upcomingSceheduledTimes`, `taxIds`).

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/recurring/invoices/aut_8a2c4e91-3b5d-4f7c-9e1a-2d4b6c8e0a1f?related=nextScheduledTime&related=upcomingSceheduledTimes"
```

## Actualizar factura recurrente

`PUT /{companyId}/recurring/invoices/{id}` sustituye **el contenido
completo** de la recurrente. No es un PATCH.

Casos típicos:

- Cambiar la plantilla (`action.template`): por ejemplo, subir el
  importe de la suscripción mensual. Los documentos ya emitidos no se
  ven afectados; solo las próximas ejecuciones generan facturas con el
  nuevo importe.
- Cambiar la frecuencia (`trigger.rrule`): el calendario se recalcula
  desde `dtstart`.
- Activar o desactivar (`enabled`): equivale a pausar/reanudar sin
  perder la configuración.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '@recurringInvoice.json' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/recurring/invoices/aut_8a2c4e91-3b5d-4f7c-9e1a-2d4b6c8e0a1f"
```

## Borrar factura recurrente

`DELETE /{companyId}/recurring/invoices/{id}` elimina la recurrente. Las
facturas ya emitidas por la recurrente **se conservan**. Solo se cancela
el calendario de ejecuciones futuras.

Si quieres pausar temporalmente sin perder configuración, usa
`enabled: false` en lugar de borrar.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/recurring/invoices/aut_8a2c4e91-3b5d-4f7c-9e1a-2d4b6c8e0a1f"
```

## Actualizar etiquetas

`PUT /{companyId}/recurring/invoices/{id}/tags` reemplaza el conjunto
de etiquetas sin tocar el resto del contenido.

###### Ejemplo de request JSON

```json
{
  "tags": ["suscripcion", "premium"]
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '{"tags":["suscripcion","premium"]}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/recurring/invoices/aut_8a2c4e91-3b5d-4f7c-9e1a-2d4b6c8e0a1f/tags"
```

## Generar muestra en PDF

`PUT /{companyId}/recurring/invoices/{id}/pdf` genera una URL temporal
con una **muestra** de la factura que se emitiría, en PDF. Es solo una
previsualización: no crea ninguna factura real ni avanza el calendario.

**Parámetros del body:**

- `mode` (opcional) — `"attachment"` (URL para descargar como adjunto,
  por defecto), `"inline"` (visualización en línea) o `"print"` (URL
  HTML que carga el PDF y lo envía a imprimir).

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '{"mode":"inline"}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/recurring/invoices/aut_8a2c4e91-3b5d-4f7c-9e1a-2d4b6c8e0a1f/pdf"
```

## Adjuntos

Una factura recurrente puede tener adjuntos vinculados (p. ej. un
contrato base). Estos adjuntos viajan en cada factura generada por la
recurrente.

### Listar adjuntos

`GET /{companyId}/recurring/invoices/{id}/attachments` devuelve los
adjuntos vinculados.

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/recurring/invoices/aut_8a2c4e91-3b5d-4f7c-9e1a-2d4b6c8e0a1f/attachments"
```

### Vincular adjuntos

`POST /{companyId}/recurring/invoices/{id}/attachments` añade adjuntos
a partir de uploads previos.

**Parámetros del body:**

- `uploadIds` — array de identificadores `upl_<uuid>`. Mínimo uno.

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"uploadIds":["upl_6f8e9b42-1d3c-4a7f-b2e4-9c1d5e7f3a6b"]}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/recurring/invoices/aut_8a2c4e91-3b5d-4f7c-9e1a-2d4b6c8e0a1f/attachments"
```

### Eliminar un adjunto

`DELETE /{companyId}/recurring/invoices/{id}/attachments/{attachmentIndex}`
elimina un adjunto. `attachmentIndex` es la posición en el array
(basada en cero).

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/recurring/invoices/aut_8a2c4e91-3b5d-4f7c-9e1a-2d4b6c8e0a1f/attachments/0"
```

## Recomendaciones

- **Empieza con `enabled: false`** mientras pruebas: la recurrente
  queda configurada sin generar facturas.
- **Usa `related=nextScheduledTime`** al consultar para saber cuándo
  se ejecutará por próxima vez. Y `upcomingSceheduledTimes` para ver
  las siguientes 5 fechas (útil para validar que el `rrule` resuelve
  al calendario que esperas).
- **Para suscripciones que cambian de precio**, actualiza la plantilla
  (`action.template.main.lines[].unitPrice`) en cualquier momento. Las
  ejecuciones futuras usan el nuevo precio sin tocar las pasadas.
- **`dtstart` en el futuro** es válido y útil para programar
  recurrentes que arranquen más adelante.
- **Para pausar temporalmente**, prefiere `enabled: false` sobre
  borrado. Conserva todo el calendario y la configuración.

## Errores comunes

- `400 ValidationError` — `rrule` con combinación inválida (`until` y
  `count` a la vez, `bymonthday > 28` con `freq: monthly`, `bysetpos`
  sin `byweekday`...).
- `400 ValidationError` — `daysAdvance != 0` (reservado, debe quedar
  en cero).
- `400 ValidationError` — `action.template.type` distinto de `invoice`.
  Hoy es el único valor soportado.
- `409 Conflict` — `series` o `docNumber` de la plantilla en conflicto
  con otra factura emitida.

Ver [Errores y validaciones](../guides/errors.md) para el formato
general.

## Referencia exhaustiva

Esta página cubre los matices funcionales y los casos típicos. Para la
referencia exhaustiva de todos los campos del body y la respuesta,
consulta el [Swagger UI](https://www.facturadirecta.com/api) o el
[openapi crudo](https://app.facturadirecta.com/openapi.json).

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| GET | `/{companyId}/recurring/invoices` | `getRecurringInvoices` | `invoices:read`, `recurring:read` | Lista de facturas de venta recurrentes |
| GET | `/{companyId}/recurring/invoices/{id}` | `getRecurringInvoice` | `invoices:read`, `recurring:read` | Obtener una factura de venta recurrente |
| GET | `/{companyId}/recurring/invoices/{id}/attachments` | `getRecurringInvoiceAttachments` | `invoices:read`, `recurring:read` | Listar adjuntos de una factura recurrente |
| POST | `/{companyId}/recurring/invoices` | `createRecurringInvoice` | `invoices:write`, `recurring:write` | Crea factura de venta recurrente |
| POST | `/{companyId}/recurring/invoices/{id}/attachments` | `addRecurringInvoiceAttachments` | `invoices:write`, `recurring:write` | Vincular adjuntos a una factura recurrente |
| PUT | `/{companyId}/recurring/invoices/{id}` | `updateRecurringInvoice` | `invoices:write`, `recurring:write` | Actualizar factura recurrente |
| PUT | `/{companyId}/recurring/invoices/{id}/pdf` | `createRecurringInvoicePDF` | `invoices:read`, `recurring:read` | Generar muestra de la factura de venta recurrente en formato PDF |
| PUT | `/{companyId}/recurring/invoices/{id}/tags` | `updateRecurringInvoiceTags` | `invoices:write`, `recurring:write` | Actualizar etiquetas de factura de venta recurrente |
| DELETE | `/{companyId}/recurring/invoices/{id}` | `deleteRecurringInvoice` | `invoices:write`, `recurring:write` | Borrar factura de venta recurrente |
| DELETE | `/{companyId}/recurring/invoices/{id}/attachments/{attachmentIndex}` | `deleteRecurringInvoiceAttachment` | `invoices:write`, `recurring:write` | Eliminar adjunto de una factura recurrente |

## Scopes

- **`invoices:read`** — Lectura de facturas.
- **`invoices:write`** — Modificación de facturas.
- **`recurring:read`** — Lectura de documentos recurrentes.
- **`recurring:write`** — Modificación de documentos recurrentes.
