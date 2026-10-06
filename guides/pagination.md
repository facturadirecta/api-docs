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
- **Cursor paginado** (`{ items, hasMore, nextCursor }`) — para secuencias
  temporales largas (eventos de webhook, actividad, versiones). El cliente
  avanza pasando el cursor que recibe con cada página.
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

- `GET /{companyId}/activity` (`getActivity`), el
  [registro de actividad](../sections/activity.md).
- `GET /{companyId}/<recurso>/{id}/versions`, las
  [versiones de un documento](./document-versions.md).
- `GET /{companyId}/webhooks/events` (`listPublicWebhookEvents`), los
  [eventos de webhook](../sections/webhooks.md).

### Parámetros

- **`limit`** — número máximo de resultados. Máximo `100`.
- **`cursor`** — opcional. Para la primera página se omite. Para las
  siguientes, se envía el `nextCursor` que devolvió la página anterior,
  **tal cual se recibió**.

### Respuesta

```json
{
  "items": [ { "...": "..." } ],
  "hasMore": true,
  "nextCursor": "NDgyODc3"
}
```

- **`items`** ordenados de más reciente a más antiguo. La actividad y las
  versiones admiten además el orden inverso con `sortBy=timestamp`.
- **`hasMore`** indica si quedan más resultados detrás de esta página.
- **`nextCursor`** es el cursor de la página siguiente, o `null` cuando
  `hasMore` es `false`.
- **No hay `pagination.total`.** En este patrón no se devuelve el total.

### Cómo paginar

1. Primera petición sin `cursor`:
   ```
   GET /{companyId}/activity?limit=100
   ```
2. Procesa `items`.
3. Si `nextCursor` no es `null`, pásalo en la siguiente petición con los
   mismos filtros y el mismo `sortBy`:
   ```
   GET /{companyId}/activity?limit=100&cursor=NDgyODc3
   ```
4. Repite hasta que `nextCursor` sea `null` (o, lo que es lo mismo,
   `hasMore` sea `false`).

### Notas

- **El cursor es opaco.** Es una cadena que identifica la posición de la
  última fila de la página; su formato no forma parte del contrato y
  puede cambiar. No lo construyas a partir de los datos del listado (por
  ejemplo, del `id` del último elemento) ni lo interpretes: un valor que
  no haya devuelto la API responde `400`.
- **El cursor marca una posición, no una consulta.** Si cambias los
  filtros o el `sortBy` entre dos páginas, la petición sigue siendo
  válida, pero la paginación continúa desde esa posición con la nueva
  consulta. Para recorrer un listado completo, mantén los mismos
  parámetros en todas las páginas.
- Para reanudar una sincronización desde un punto conocido (por ejemplo,
  tras una caída), guarda el `nextCursor` de la última página que
  procesaste con éxito y úsalo como `cursor` en la próxima ejecución. En
  la actividad y las versiones, pide además `sortBy=timestamp` para
  avanzar hacia lo más reciente.

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

En el patrón de cursor, los eventos de webhook tienen un orden fijo:
`created_at` descendente, sin `sortBy`. La actividad y las versiones
admiten `sortBy=-timestamp` (por defecto, lo más reciente primero) y
`sortBy=timestamp`.

## Recomendaciones

- **Pide solo lo que necesitas.** `limit=50` cubre la mayoría de casos
  de UI y reduce latencia.
- **Para sincronizaciones masivas en patrón A**, filtra por
  `minModificationDate` con el timestamp de tu última sincronización
  exitosa en vez de paginar el recurso completo desde cero.
- **Para la actividad, las versiones y los eventos de webhook (patrón
  B)**, persiste el cursor de la última página procesada y úsalo en la
  siguiente sesión: no recorras todo el historial cada vez.
- **Comprueba `pagination.total` antes de paginar** en el patrón A; si
  es pequeño, una sola petición basta.
- **No asumas orden estable** en el patrón A si no envías `sortBy`. Si
  necesitas consistencia entre páginas, fíjalo explícitamente.
