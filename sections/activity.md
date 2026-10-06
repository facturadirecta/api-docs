---
title: Actividad
audience:
  - developers
status: draft
---

# Actividad

El **registro de actividad** es el historial de lo que pasa en la
empresa: altas, modificaciones y eliminaciones de documentos, envíos por
email y su entrega, accesos de los usuarios, sincronizaciones bancarias,
envíos a la AEAT y a FACe, ejecuciones de tareas automáticas, cambios de
ajustes… Es la misma información que muestra la página **Actividad** de
la aplicación.

La API expone una sola operación de **lectura con paginación por
cursor**. Sirve para auditar quién hizo qué y cuándo, para ver el
historial de un documento concreto o para llevar a otro sistema la
actividad nueva desde la última consulta.

Cada registro trae un núcleo común con forma fija (fecha, tipo, usuario,
descripción, elemento afectado y origen de la petición) y un objeto
`details` con algunos datos propios de su tipo. El contenido de los
documentos no viaja en la actividad: qué campos cambió cada modificación, sus
valores y la copia de cada momento están en las
[versiones del documento](../guides/document-versions.md).

## Permisos

- El scope es `activity:read`.
- Con OAuth, además, el rol del usuario en la empresa necesita el permiso
  de **actividad completo** (ver la actividad de todos), como los roles
  *Administrador* y *Asesor*. Los roles que solo ven su propia actividad
  (en los predefinidos, *Sólo lectura*, *Ventas*, *Compras* y *Ventas y
  compras*) reciben un `403`: la API no filtra la actividad por usuario.
- Una API key accede si tiene el scope. Al crearla, solo puede concederle
  `activity:read` quien tiene el permiso de actividad completo.
- Por ahora, este permiso no se puede conceder a las aplicaciones
  conectadas (servidor MCP y CLI).

## Estructura de un registro

```json
{
  "id": 482913,
  "timestamp": "2026-09-15T08:42:17.351Z",
  "type": "send",
  "subtype": null,
  "username": "ana.garcia@ejemplo.es",
  "description": "Enviado",
  "document": {
    "id": "inv_7c1e2f40-5a8b-4d3c-9e6f-2b1a0c9d8e7f",
    "type": "invoice",
    "title": "F2026-0123",
    "archived": false
  },
  "request": {
    "ip": "88.12.34.56",
    "userAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)"
  },
  "details": {
    "actionSource": "user",
    "to": "pedidos@clienteejemplo.es"
  }
}
```

| Campo | Tipo | Significado |
|---|---|---|
| `id` | entero | Identificador del registro. |
| `timestamp` | string | Fecha y hora de la actividad (ISO 8601, UTC). |
| `type` | string | Tipo de actividad (ver [Tipos de actividad](#tipos-de-actividad)). |
| `subtype` | string \| null | Subtipo, en los tipos que lo tienen: por ejemplo `voided` en una modificación que anula una factura, o `psd2` en una sincronización bancaria. |
| `username` | string \| null | Usuario (email) que hizo la acción. Las acciones automáticas aparecen con un usuario de sistema de FacturaDirecta, como `ami@facturadirecta.com` en las tareas automáticas. |
| `description` | string | Texto legible en español, el mismo que muestra la página de actividad. |
| `document` | objeto \| null | Elemento al que se refiere la actividad. `null` en la actividad de la empresa, como los accesos o los cambios de ajustes. |
| `request` | objeto \| null | Origen de la petición (`ip`, `userAgent`), cuando se conoce. |
| `details` | objeto | Datos propios del tipo de actividad. |

### El elemento afectado: `document`

- **`id`** — identificador del elemento: una factura (`inv_…`), un
  contacto (`con_…`), un banco, una tarea automática, un cobro…
- **`type`** — tipo del documento (`invoice`, `bill`, `contact`,
  `product`…). Es `null` cuando el elemento no es un documento.
- **`title`** — título del documento, por ejemplo el número de una
  factura o el nombre de un contacto.
- **`archived`** — `true` si el documento está eliminado.

Con el `id` puedes pedir el detalle en el recurso correspondiente
([facturas](./invoices.md), [contactos](./contacts.md)…), con el scope
de ese recurso.

### Datos del tipo: `details`

`details` no copia el registro interno: es una selección de campos de
cada tipo. Incluye identificadores de otros elementos, resultados
(estados, contadores, errores) y datos de la acción (formato, canal,
destinatarios de un envío). Un campo que el registro no tiene no
aparece, y la lista puede ampliarse con campos nuevos sin aviso, así
que tu integración debe ignorar los campos que no conozca.

Muchos tipos devuelven un objeto vacío. Las altas y modificaciones de
documentos no traen el contenido del documento: qué cambió en cada una se
consulta en las [versiones del documento](../guides/document-versions.md), donde
el `id` del registro de actividad es el identificador de la versión.

| Tipo | Campos de `details` |
|---|---|
| `inserted` | `automationId` (tarea automática que creó el documento), `importSource` |
| `comment_inserted`, `comment_updated`, `comment_archived` | `commentId` |
| `file` | `format` (`pdf`, `facturae`, `sepa`, `boefile`) |
| `portalView` | `action` (`view`: consulta, `pdf`: descarga del PDF) |
| `portalPaymentFinalizeFailed` | `transactionId` (el cobro recibido), `error` (por qué no se pudo emitir la factura) |
| `deliveryReceiptSigned` | `signedAt` |
| `send`, `send_queued` | `actionSource`, `to`, `cc`, `bcc` |
| `send_delivered`, `send_dropped`, `send_complained`, `send_failed_temporary`, `send_failed_permanent` | `recipient`, `message` |
| `smtpCustomServerAutoDisabled` | `serverId`, `serverName`, `reason` |
| `bankSync` | `manualSync`, `processedTransactions`, `loadedTransactions`, `updatedTransactions`, `repeatedTransactions`, `ignoredTransactions`, `autoReconciledTransactions` |
| `bankSyncWarning` | `manualSync`, `message` |
| `ticketbai_accepted`, `ticketbai_rejected` | `diputacionForal` |
| `ticketbai_error` | `diputacionForal`, `error` |
| `verifactu_send_aeat` | `status`, `invoiceIds`, `error` |
| `verifactu_send_invoice` | `operation`, `status`, `errorCode`, `errorDescription`, `error` |
| `verifactu_send_requerimiento` | `reference`, `totalRecords`, `totalBatches` |
| `verifactu_export_records` | `startDate`, `endDate`, `totalRecords` |
| `verifactu_export_events` | `startDate`, `endDate`, `totalEvents` |
| `face_send_invoice` | `submissionId`, `registryCode`, `environment`, `trigger`, `sendStatus` |
| `face_adopt_registration` | `submissionId`, `registryCode`, `environment`, `trigger`, `registrationOrigin` |
| `face_refresh_status` | `submissionId`, `registryCode`, `environment`, `trigger`, `invoiceStatus`, `cancellationStatus` |
| `face_cancel_invoice` | `submissionId`, `registryCode`, `environment`, `trigger` |
| `accounting_export` | `format`, `startDate`, `endDate`, `elements` |
| `automation` | `scheduledTime`, `createdDocumentId`, `warnings` |
| `automationError` | `scheduledTime`, `errors` |
| `automationInfo` | `reason`, `nextScheduledTime`, `createdDocumentId` |
| `automationAutoDeactivated` | `reason` |
| `automationReport` | `periodStart`, `periodEnd`, `summary` |
| `updateSettings` | `reason` |
| `archiveCompany` | `reason` |
| `certificateExpirationEmail` | `validityNotAfter`, `daysUntilExpiration` |
| `usageMeterReset` | `meter`, `month` |
| `usageQuotaWarning` | `meter`, `period`, `month`, `year`, `threshold`, `quota`, `count` |
| `usageCreditGrant`, `usageCreditPurchase`, `usageCreditChargeback`, `usageCreditAdjustment` | `amountCents`, `balanceAfterCents` |
| `webhook` | `url`, `success`, `responseStatus` |
| `webhook_endpoint_*`, `webhook_secret_rotated` | `endpointId`, `name`, `url`, `events`, `changedFields`, `failureCount` |

Las sincronizaciones bancarias del sistema anterior tienen sus propios
tipos y devuelven los mismos campos que `bankSync` (más `error`) y
`bankSyncWarning`. El resto de tipos devuelve un objeto vacío.

<!-- review: selección completa por tipo en PUBLIC_ACTIVITY_DETAILS (publicActivity.ts); verificado 2026-10-02 -->

### Tipos de actividad

Los valores de `type` son los del filtro `type` de la operación (la
lista completa está en el openapi). Los más habituales:

- **Documentos:** `inserted` (alta), `updated` (modificación, con
  `subtype` cuando es una acción concreta como `voided`, `unvoided`,
  `addAttachments` o `updateTags`), `archived` (eliminación),
  `unarchived` (recuperación), `comment_inserted`, `file` (generación
  de un PDF, Facturae o remesa).
- **Envíos por email:** `send`, `send_queued`, `send_delivered`,
  `send_failed_temporary`, `send_failed_permanent`, `send_dropped`,
  `send_complained`, y `portalView` para las consultas del documento en
  el portal de cliente. `portalPaymentFinalizeFailed` indica que se ha
  cobrado por el portal una factura provisional que no se ha podido
  emitir.
- **Usuarios y empresa:** `access` (entrada de un usuario),
  `updateSettings`, `leaveCompany`.
- **Bancos:** `bankSync`, `bankSyncWarning` (`subtype` indica el canal:
  `psd2` para la conexión automática con el banco y `paymentGateway` para
  las pasarelas de pago).
- **Fiscalidad:** `verifactu_send_aeat`, `verifactu_send_invoice`,
  `ticketbai_accepted`, `ticketbai_rejected`, `ticketbai_error`,
  `face_send_invoice`, `face_refresh_status`, `face_cancel_invoice`.
- **Tareas automáticas:** `automation`, `automationError`,
  `automationInfo`, `automationReport`.
- **Webhooks:** `webhook` (entrega), `webhook_endpoint_created` y el
  resto de cambios de endpoints.

Los registros antiguos pueden tener tipos que ya no se generan. Trata
`type` como un valor abierto: si no lo conoces, usa `description`.

## Operaciones

- [Listar la actividad](#listar-la-actividad)

## Listar la actividad

`GET /{companyId}/activity` devuelve la actividad con **paginación por
cursor** (ver el patrón B en la [guía de paginación](../guides/pagination.md)).

**Respuesta:** `{ items: Activity[], hasMore: boolean, nextCursor: string | null }`.
Para pedir la página siguiente, pasa `nextCursor` en `cursor`; cuando es
`null`, no hay más.

**Parámetros de consulta:**

- `limit` — número máximo de resultados, entre `1` y `100`. Por
  defecto, `50`.
- `cursor` — el `nextCursor` de la página anterior, tal cual se recibió.
  Devuelve la página que va detrás de ella en el orden pedido. Para la
  primera página, se omite. Es un valor opaco: uno que no haya devuelto
  la API responde `400`.
- `sortBy` — `-timestamp` (por defecto) devuelve primero lo más
  reciente; `timestamp`, lo más antiguo. A igual fecha, se ordena por
  `id`.
- `type` — tipos de actividad a incluir. Se repite para indicar varios
  (`type=inserted&type=archived`) y se incluye la actividad de
  cualquiera de ellos.
- `username` — usuarios (email) que registraron la actividad. Admite
  varios, como `type`.
- `documentId` — identificador del elemento al que se refiere la
  actividad. Devuelve el historial de ese elemento.
- `documentType` — tipos del documento afectado (`invoice`, `bill`,
  `contact`…). Admite varios.
- `minDate` / `maxDate` — fecha y hora mínima y máxima, ambas incluidas,
  en formato ISO 8601 (`YYYY-MM-DDTHH:mm:ss.sssZ`). Codifica el valor en
  la URL (`:` como `%3A`).

**Parámetros globales aceptados:**

Acepta además el header `accept-version`. Ver
[Autenticación](../guides/authentication.md).

> Este endpoint **no** acepta `offset` ni los filtros estándar
> `minCreationDate`/`maxCreationDate`/`minModificationDate`/`maxModificationDate`.

###### Ejemplo de respuesta

```json
{
  "items": [
    {
      "id": 482913,
      "timestamp": "2026-09-15T08:42:17.351Z",
      "type": "send",
      "subtype": null,
      "username": "ana.garcia@ejemplo.es",
      "description": "Enviado",
      "document": {
        "id": "inv_7c1e2f40-5a8b-4d3c-9e6f-2b1a0c9d8e7f",
        "type": "invoice",
        "title": "F2026-0123",
        "archived": false
      },
      "request": { "ip": "88.12.34.56", "userAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)" },
      "details": { "actionSource": "user", "to": "pedidos@clienteejemplo.es" }
    },
    {
      "id": 482877,
      "timestamp": "2026-09-15T07:00:04.120Z",
      "type": "bankSync",
      "subtype": "psd2",
      "username": "ami@facturadirecta.com",
      "description": "Sincronización bancaria realizada",
      "document": { "id": "<id-banco>", "type": "bank", "title": "Cuenta principal", "archived": false },
      "request": null,
      "details": {
        "processedTransactions": 12,
        "loadedTransactions": 9,
        "repeatedTransactions": 3,
        "ignoredTransactions": 0
      }
    }
  ],
  "hasMore": true,
  "nextCursor": "NDgyODc3"
}
```

`<id-banco>` es un placeholder: cada empresa tiene sus propios
identificadores.

###### Copy as cURL

Primera página:

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/activity?limit=100"
```

Siguiente página, con el `nextCursor` de la anterior:

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/activity?limit=100&cursor=NDgyODc3"
```

Historial de una factura, del más antiguo al más reciente:

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/activity?documentId=inv_7c1e2f40-5a8b-4d3c-9e6f-2b1a0c9d8e7f&sortBy=timestamp"
```

Documentos eliminados en septiembre:

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/activity?type=archived&minDate=2026-09-01T00%3A00%3A00.000Z&maxDate=2026-09-30T23%3A59%3A59.999Z"
```

## Recomendaciones

- **Para llevar la actividad nueva a otro sistema**, pide en orden
  ascendente (`sortBy=timestamp`) y guarda el `nextCursor` de la última
  página que procesaste. En la siguiente consulta, pásalo en `cursor`:
  recibes solo lo posterior, sin repetir ni saltar registros.
- **Para auditar un documento**, filtra por `documentId` en lugar de
  recorrer toda la actividad. Para ver qué valores cambiaron y recuperar
  datos, usa las [versiones del documento](../guides/document-versions.md).
- **Acota con `minDate`/`maxDate`** las consultas de históricos largos:
  la actividad de una empresa crece sin límite. Con `type`, añade
  también las fechas si buscas un tipo que no es reciente; si no, la
  consulta repasa toda la actividad posterior hasta encontrarlo. Una
  consulta que tarda demasiado se corta con un `422`.
- **No dependas de `description` para automatizar.** Es texto para
  personas y puede cambiar; usa `type`, `subtype` y `details`.

## Errores comunes

- `400 ValidationError` — `cursor` no es un cursor devuelto por la API,
  `minDate`/`maxDate` no son fechas ISO 8601, `type` o `sortBy` tienen
  un valor desconocido, o `limit` está fuera de `1`–`100`.
- `422 Unprocessable Entity` (`code: "activity_query_timeout"`) — la
  consulta ha tardado demasiado. Acota la búsqueda con `minDate` y
  `maxDate` o con más filtros.
- `403 Forbidden` (`code: "token_scope_missing"`) — falta el scope
  `activity:read`.
- `403 Forbidden` (`code: "user_role_insufficient"`) — el rol del
  usuario no tiene el permiso de actividad completo.

Ver [Errores y validaciones](../guides/errors.md) para el formato
general.

## Referencia exhaustiva

Esta página cubre los matices funcionales y los casos típicos. Para el
contrato completo del registro, consulta el
[Swagger UI](https://www.facturadirecta.com/api) o el
[openapi crudo](https://app.facturadirecta.com/openapi.json).

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| GET | `/{companyId}/activity` | `getActivity` | `activity:read` | Registro de actividad de la empresa |

## Scopes

- **`activity:read`** — Lectura del registro de actividad de la empresa.
