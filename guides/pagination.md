---
title: Paginación y filtros estándar
audience:
  - developers
status: draft
---

# Paginación y filtros estándar

La API pública usa **tres patrones de listado** distintos, no uno. Cada
operación que devuelve una lista escoge el patrón que mejor encaja con el
recurso. La página de cada recurso indica cuál aplica:

- **Offset paginado** (`{ pagination, items }`) — el más común. Para
  recursos navegables por páginas.
- **Cursor paginado** (`{ items, hasMore }`) — para secuencias temporales
  largas (eventos de webhook). El cliente avanza con un timestamp.
- **Sin paginación** (`{ items }`) — para catálogos pequeños y listas
  anidadas, donde devolver todo el conjunto de una vez es razonable.

## Patrón A — Offset paginado

Aplica a la mayoría de recursos: `contacts`, `products`, `invoices`,
`recurring`, `estimates`, `deliveryNotes`, `bills`, `payrolls`, `banks`,
`statements`, `paymentMethods`, `journal`, `inbox`.

### Parámetros

- **`offset`** — posición de inicio (basada en cero). Por defecto `0`.
- **`limit`** — número máximo de resultados. Entre `1` y `500`. Si no se
  indica, el servidor aplica un valor por defecto razonable.

Ejemplo:

```
GET /{companyId}/contacts?offset=100&limit=50
```

devuelve hasta 50 contactos a partir del 101º (índice 100).

### Respuesta

```json
{
  "pagination": { "offset": 0, "limit": 50, "total": 124 },
  "items": [
    { "...": "..." }
  ]
}
```

- **`pagination.offset`** y **`pagination.limit`** repiten los valores
  efectivamente aplicados (útil cuando no envías uno y quieres saber el
  default).
- **`pagination.total`** es el total de elementos que cumplen los filtros
  aplicados, **no** el total absoluto del recurso.
- **`items`** es siempre el nombre del array.

Para recorrer el resultado completo, incrementa `offset` en pasos de
`limit` hasta que `offset + items.length >= total`.

## Patrón B — Cursor paginado

Aplica a:

- `GET /{companyId}/webhooks/events` (`listPublicWebhookEvents`).

### Parámetros

- **`limit`** — número máximo de resultados. Máximo `100`.
- **`cursor`** — opcional. Para la primera página se omite. Para páginas
  posteriores, se envía el timestamp `created_at` del **último elemento**
  devuelto en la página anterior.

### Respuesta

```json
{
  "items": [ { "...": "..." } ],
  "hasMore": true
}
```

- **`items`** ordenados de más reciente a más antiguo (`created_at` desc).
- **`hasMore`** indica si quedan más resultados que la página actual.
- **No hay `pagination.total`.** En este patrón no se devuelve el total.

### Cómo paginar

1. Primera petición sin `cursor`:
   ```
   GET /{companyId}/webhooks/events?limit=100
   ```
2. Procesa `items` (típicamente del más reciente al más antiguo).
3. Si `hasMore` es `true`, toma el `created_at` del **último elemento**
   del array y úsalo como `cursor` en la siguiente petición:
   ```
   GET /{companyId}/webhooks/events?limit=100&cursor=<created_at>
   ```
4. Repite hasta que `hasMore` sea `false`.

### Notas

- El `cursor` es un timestamp en formato ISO 8601 UTC (`YYYY-MM-DDTHH:mm:ss.sssZ`),
  no un identificador opaco. Coincide con el campo `created_at` de los
  eventos.
- Para reanudar una sincronización desde un punto conocido (por ejemplo,
  reanudar tras una caída), guarda el `created_at` del último evento que
  procesaste con éxito y úsalo como `cursor` en la próxima ejecución.

## Patrón C — Sin paginación

Aplica a operaciones que devuelven el conjunto completo sin paginar:

- **Adjuntos de documento:** `getInvoiceAttachments`,
  `getRecurringInvoiceAttachments`, `getEstimateAttachments`,
  `getDeliveryNoteAttachments`, `getBillAttachments`,
  `getPayrollAttachments`.
- **Catálogos de empresa:** `getSalesTaxes`, `getPurchasesTaxes`,
  `getThemes`, `getSeries`.
- **Listas de gestión:** `listPublicWebhookEndpoints`,
  `listPublicApiKeys`.

### Respuesta

```json
{
  "items": [ { "...": "..." } ]
}
```

No hay `pagination` ni `hasMore`. Asume que la cantidad de elementos es
manejable: típicamente decenas, no miles. Si un recurso de este tipo
crece más allá de lo razonable, su endpoint se migrará a uno de los
patrones paginados.

## Filtros estándar de fecha

Sobre los recursos de tipo documento (`invoices`, `bills`, `estimates`,
`deliveryNotes`, `payrolls`, `recurring`), las listas con paginación
offset aceptan además cuatro filtros temporales comunes, en formato
ISO 8601 UTC:

- **`minCreationDate`** / **`maxCreationDate`** — rango por fecha de creación del documento en FacturaDirecta.
- **`minModificationDate`** / **`maxModificationDate`** — rango por fecha de última modificación.

Formato esperado: `YYYY-MM-DDTHH:mm:ss.sssZ` (UTC). Ejemplo:

```
GET /{companyId}/invoices?minCreationDate=2026-01-01T00:00:00.000Z&maxCreationDate=2026-04-01T00:00:00.000Z
```

Estos filtros existen para detectar cambios desde un timestamp conocido
(sincronizaciones incrementales). Cada recurso documento añade además
sus propios filtros de fecha semánticos (por ejemplo `minDate`, `maxDate`
en facturas y presupuestos, sobre la fecha del documento, no la de
creación).

## Ordenación

Los recursos exponen un parámetro `sortBy` cuando soportan ordenación
explícita. Los valores admitidos dependen del recurso y se documentan en
su página. Sin `sortBy`, cada recurso aplica un orden por defecto razonable
(habitualmente fecha descendente para documentos, alfabético para
catálogos).

En el patrón de cursor (eventos de webhook), el orden es fijo: `created_at`
descendente. No admite `sortBy`.

## Recomendaciones

- **Pide solo lo que necesitas.** `limit=50` cubre la mayoría de casos
  de UI y reduce latencia.
- **Para sincronizaciones masivas en patrón A**, filtra por
  `minModificationDate` con el timestamp de tu última sincronización
  exitosa en vez de paginar el recurso completo desde cero.
- **Para eventos de webhook (patrón B)**, persiste el `created_at` del
  último evento procesado y úsalo como cursor en la siguiente sesión:
  no recorras todo el log cada vez.
- **Comprueba `pagination.total` antes de paginar** en el patrón A; si
  es pequeño, una sola petición basta.
- **No asumas orden estable** en el patrón A si no envías `sortBy`. Si
  necesitas consistencia entre páginas, fíjalo explícitamente.
