---
title: Bandeja de entrada
audience:
  - developers
status: draft
---

# Bandeja de entrada

La bandeja de entrada (`inbox`) es el flujo para **incorporar documentos
externos** (PDFs de facturas de proveedores, tickets escaneados,
nóminas, ficheros Facturae...) y convertirlos automáticamente en
gastos, tickets o nóminas con datos pre-rellenados.

El flujo conceptual tiene cuatro pasos:

1. **Subir** el archivo binario con `POST /uploads` y obtener un `upl_<uuid>`.
2. **Crear** el item de bandeja con `POST /inbox` referenciando el upload.
   El sistema escanea el documento en background.
3. **Esperar** (o suscribirse al webhook `inbox.scanned`) hasta que
   `status` pase a `scanned`. La extracción aparece en `scan.extraction`.
4. **Proponer** un documento real con `POST /inbox/{id}/proposeBill`
   (o `proposeTicket`, o `proposePayroll`). Devuelve un prototipo
   listo para enviar a `POST /bills` (o `/payrolls`) con un campo
   adicional `fromInbox: { taskId }` que vincula los dos documentos.

Los `propose*` **no crean nada en BD**: son funciones puras que mapean
los datos extraídos al shape esperado por la API de creación. Permite
al cliente (o a un agente) revisar y ajustar la propuesta antes de
materializarla.

> En los ejemplos de esta página, los UUIDs (`tas_…`, `upl_…`,
> `con_…`) son ilustrativos. Cada empresa tiene los suyos; sustitúyelos
> por los identificadores reales que devuelve la API.

## El prefijo es `tas_`, no `inb_`

Los items de la bandeja **se identifican con prefijo `tas_<uuid>`**
porque internamente son "tareas" (`task`) pendientes de procesar. Las
tareas existen también para otros usos internos; la bandeja es solo
una vista de las que provienen de subidas, emails o sincronizaciones
externas. Para la API pública, todos los `inbox.id` son `tas_*`.

## Estados del item

`status` describe el escaneo automático:

- **`pending`** — el item se acaba de crear, el escaneo no ha terminado.
  `scan` es `null`.
- **`scanned`** — el escaneo terminó con éxito. `scan` y
  `scan.extraction` están poblados.
- **`scan_failed`** — el escaneo terminó sin datos extraíbles (PDF
  ilegible, imagen sin texto, etc.). `scan.completedAt` y `scan.mimeType`
  están rellenos pero `scan.extraction` puede ser nulo o muy parcial.

## Orígenes (`source`)

Un item puede llegar a la bandeja por cuatro vías:

- **`upload`** — subido vía API pública (`POST /inbox` con un upload
  previo). Es el único origen que esta API permite generar
  directamente.
- **`inbox`** — recibido por email vía la dirección de Mailgun de la
  empresa.
- **`googleDrive`** — sincronizado automáticamente desde una carpeta
  de Google Drive vinculada.
- **`scanner`** — subido manualmente desde la pantalla de Tareas en
  la interfaz.

Todos los orígenes generan items con el mismo schema y se procesan
con la misma lógica de escaneo y propuesta.

## Adjuntos del item

`content.attachments` es la lista de ficheros del item. Por lo general
es un único adjunto (el subido inicialmente), pero algunos orígenes
adjuntan varios (por ejemplo un email con factura PDF + cuerpo HTML).

Cada adjunto tiene `url` S3 directa (permanente mientras el item no se
borre físicamente), `name`, `size`, `contentType` (MIME) y opcionalmente
`contentSubtype`:

- **`facturae`** — el adjunto es un fichero Facturae XML. El sistema lo
  parsea con un extractor nativo (no LLM), por lo que la extracción
  es exacta y rápida.
- **`inboxEmailBody`** — cuerpo del email recibido (cuando `source` es
  `inbox`).

## Extracción y `scan`

Cuando `status` es `scanned`, `content.scan` contiene metadatos del
escaneo y los datos estructurados:

- `scan.completedAt` — cuándo terminó.
- `scan.docType` — tipo de documento detectado (p. ej. `IMAGE`, `PDF`).
- `scan.mimeType` — MIME del archivo.
- `scan.text` — texto plano extraído. **Solo aparece en
  `GET /inbox/{id}`**, no en el listado (optimización de tamaño).
- `scan.extraction` — datos estructurados. El shape depende del tipo
  de documento detectado: `InboxExtractionInvoice` para facturas y
  tickets, `InboxExtractionPayroll` para nóminas.

La extracción contiene campos como `date`, `name` (emisor), `contact`
(NIF/CIF), `number`, `total`, `currency`, `direction` (`RECEIVE` o
`EMIT`)... La API expone estos campos como **strings sin tipar** para no
acoplar la API a la versión actual del extractor; nuevos campos pueden
aparecer sin cambio de contrato.

## Webhook `inbox.scanned`

Cuando un escaneo termina (cualquier `status` final, incluido
`scan_failed`), se dispara el evento de webhook `inbox.scanned`.
Suscribirse permite reaccionar en cuanto los datos están disponibles
en vez de hacer polling. Ver [Webhooks](./webhooks.md) para los
detalles del flujo.

## Operaciones

- [Lista de items](#lista-de-items)
- [Crear item (subir documento)](#crear-item-subir-documento)
- [Obtener un item](#obtener-un-item)
- [Archivar un item](#archivar-un-item)
- [Proponer factura de compra](#proponer-factura-de-compra)
- [Proponer ticket](#proponer-ticket)
- [Proponer nómina](#proponer-nómina)

## Lista de items

`GET /{companyId}/inbox` devuelve los items de la bandeja.

**Parámetros de consulta específicos:**

- **`status`** — filtra por estado del escaneo: `pending`, `scanned`,
  `scan_failed`.
- **`source`** — filtra por origen: `upload`, `inbox`, `googleDrive`,
  `scanner`.
- **`minDate`**, **`maxDate`** — rango por fecha de creación del item
  (ISO 8601).
- **`archived`** — controla qué items aparecen en el resultado:
  - `false` (por defecto): solo activos.
  - `true`: solo archivados.
  - `all`: ambos.

**Parámetros globales aceptados:**

Acepta además los parámetros estándar `offset`, `limit` y el header
`accept-version`. Ver [Paginación](../guides/pagination.md) y
[Autenticación](../guides/authentication.md).

**Notas:**

- `scan.text` **no se incluye** en los items del listado para evitar
  cargas grandes. Para verlo, usa `GET /inbox/{id}`.
- Por defecto, los items archivados se omiten. Esto es coherente con
  el comportamiento del `DELETE /inbox/{id}` (que archiva en lugar de
  borrar físicamente).

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/inbox?status=scanned&source=upload&limit=50"
```

## Crear item (subir documento)

`POST /{companyId}/inbox` crea un nuevo item de bandeja a partir de un
upload previo.

**Flujo de dos pasos:**

1. Subir el binario con `POST /uploads` (multipart) → recibes
   `upl_<uuid>`.
2. Crear el inbox item con este endpoint, pasando el `upl_<uuid>` en
   el campo `upload`.

**Parámetros del body:**

- **`upload`** (obligatorio) — `upl_<uuid>` devuelto por
  `POST /uploads`.
- **`title`** (opcional) — título descriptivo. Si se omite, se usa el
  nombre del archivo del upload.
- **`idempotencyKey`** (opcional) — clave de idempotencia. Reintentos
  con la misma clave devuelven el mismo item; útil para no duplicar
  cuando el cliente reintenta.

**Notas:**

- El item se crea con `status: pending` y se encola el escaneo en
  background.
- La respuesta es el item recién creado. Suscríbete al webhook
  `inbox.scanned` o haz polling sobre `status` para detectar cuándo
  termina el escaneo.

**Parámetros globales aceptados:** `accept-version`.

###### Ejemplo de request JSON

Heredado del ejemplo `default` del openapi:

```json
{
  "upload": "upl_6f8e9b42-1d3c-4a7f-b2e4-9c1d5e7f3a6b",
  "title": "factura.pdf"
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"upload":"upl_6f8e9b42-1d3c-4a7f-b2e4-9c1d5e7f3a6b","title":"factura.pdf"}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/inbox"
```

## Obtener un item

`GET /{companyId}/inbox/{id}` devuelve el detalle de un item por su
ID.

A diferencia del listado, **`scan.text` sí se incluye** (texto plano
extraído del documento).

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/inbox/tas_3a7f9c12-2d4e-4b8a-9c1f-5d6e8f0a3b2c"
```

## Archivar un item

`DELETE /{companyId}/inbox/{id}` **archiva** el item (no lo borra
físicamente). Los items archivados:

- No aparecen en el listado por defecto.
- Conservan sus adjuntos (las URLs S3 siguen siendo accesibles).
- Pueden seguir consultándose por ID si conoces el `tas_*`.
- Aparecen en listados con `archived=true` o `archived=all`.

Si vinculas un inbox item a un documento real vía `fromInbox`, el
sistema archiva el inbox automáticamente (por defecto). Ese es el
camino habitual; este endpoint cubre los casos en que quieres
archivar manualmente sin crear un documento (p. ej. spam).

**Respuesta:** `{ result: true }`.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/inbox/tas_3a7f9c12-2d4e-4b8a-9c1f-5d6e8f0a3b2c"
```

## Proponer factura de compra

`POST /{companyId}/inbox/{id}/proposeBill` toma un item escaneado y
construye un **prototipo de factura de compra** mapeando los datos
extraídos al shape de `BillWrite`. **El endpoint no crea nada en BD**:
devuelve un objeto para que el cliente lo revise, lo ajuste y lo envíe
a `POST /bills`.

**Idempotente y retry-safe.** Llamarlo varias veces sobre el mismo
item devuelve el mismo prototipo (suponiendo que la extracción no
haya cambiado).

**Parámetros del body** (todos opcionales):

- **`contactId`** — si se indica, salta el matching automático y usa
  este contacto en el prototipo.

**Respuesta:**

```json
{
  "content": { "...": "BillWrite — prototipo listo para POST /bills" },
  "matchedContact": null,
  "candidateContacts": [],
  "newContactProposal": { "...": "Contact — proto-contacto con datos extraídos" }
}
```

- **`content`** — el prototipo de bill. Su `uuid` se omite
  intencionadamente; lo asigna `POST /bills` o lo eliges tú al crear.
- **`matchedContact`** — contacto único encontrado por la API a partir
  del NIF/CIF/nombre extraídos. `null` si no se encontró ninguno o si
  hay varios candidatos.
- **`candidateContacts`** — lista de candidatos cuando el matching
  encontró varios. Vacío si hay 0 o 1 match.
- **`newContactProposal`** — proto-contacto con los datos extraídos.
  Útil cuando el matching no encontró candidatos: puedes crearlo con
  `POST /contacts` y enlazarlo al bill.

**Flujo recomendado tras la propuesta:**

1. Revisa el `content` y ajústalo si quieres (cambiar líneas, impuestos,
   añadir notas).
2. Si `matchedContact` es null:
   - O bien usa `candidateContacts` para elegir uno.
   - O bien crea uno desde `newContactProposal` con `POST /contacts`.
   - Pon el ID resultante en `content.main.contact`.
3. Llama a `POST /bills` con `{ content, fromInbox: { taskId, archive } }`:
   - `taskId` es el `tas_*` del item de la bandeja.
   - `archive: true` (por defecto) archiva el inbox tras crear el bill.
   - Esto vincula el bill al inbox; se puede ver desde la interfaz.

**Parámetros globales aceptados:** `accept-version`.

###### Ejemplo de request JSON

Sin body (la API usa la extracción tal cual):

```json
{}
```

Con contacto fijado:

```json
{
  "contactId": "con_2c4f8e21-7b3a-4d9c-9e1f-a8b7c6d5e4f3"
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/inbox/tas_3a7f9c12-2d4e-4b8a-9c1f-5d6e8f0a3b2c/proposeBill"
```

## Proponer ticket

`POST /{companyId}/inbox/{id}/proposeTicket` es equivalente a
[Proponer factura de compra](#proponer-factura-de-compra) pero
construye un prototipo con `subtype: "ticket"` (ver
[Factura de compra vs ticket](./bills.md#factura-de-compra-vs-ticket)
en la página de bills).

Útil cuando el documento escaneado es un ticket de caja (sin proveedor
identificado) en vez de una factura formal.

Misma forma de respuesta y mismo flujo de uso que `proposeBill`. El
prototipo enviado a `POST /bills` ya viene con `subtype: "ticket"`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/inbox/tas_3a7f9c12-2d4e-4b8a-9c1f-5d6e8f0a3b2c/proposeTicket"
```

## Proponer nómina

`POST /{companyId}/inbox/{id}/proposePayroll` es equivalente a las
anteriores pero construye un prototipo de **nómina** (`PayrollWrite`).
El item debe haber sido escaneado y reconocido como nómina por el
extractor (`scan.extraction` debe ser `InboxExtractionPayroll`).

Misma forma de respuesta: `{ content, matchedContact, candidateContacts,
newContactProposal }`. El contacto en este caso es el empleado.

Para crear la nómina real tras revisar el prototipo, llama a
`POST /payrolls` con `{ content, fromInbox: { taskId, archive } }`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/inbox/tas_3a7f9c12-2d4e-4b8a-9c1f-5d6e8f0a3b2c/proposePayroll"
```

## Vinculación con `fromInbox`

Al crear un bill o un payroll desde un inbox item, se incluye en el
body del create el campo opcional:

```json
{
  "content": { "...": "..." },
  "fromInbox": {
    "taskId": "tas_3a7f9c12-2d4e-4b8a-9c1f-5d6e8f0a3b2c",
    "archive": true
  }
}
```

- **`taskId`** — el `tas_*` del inbox item.
- **`archive`** (opcional, default `true`) — si `true`, el inbox se
  archiva automáticamente al crear el documento.

`fromInbox` está disponible en `POST /bills` y `POST /payrolls`. No
está disponible en `POST /invoices` (no hay flujo "subir factura de
venta a inbox" a día de hoy).

El vínculo persiste: la factura/nómina creada queda enlazada al inbox
item en sus `attachments`, y la interfaz puede mostrar el original.

## Recomendaciones

- **Suscríbete al webhook `inbox.scanned`** en vez de hacer polling
  sobre el listado. El polling funciona pero gasta cuota
  innecesariamente.
- **Usa `idempotencyKey` al crear** items si tu pipeline puede
  reintentar el `POST /inbox` (errores de red, jobs reencolados...).
  Te ahorra duplicados.
- **Llama a `propose*` solo cuando `status: scanned`**. Si lo llamas
  antes, la propuesta saldrá vacía o sin datos extraídos.
- **Para Facturae, la extracción es nativa y exacta**: usa
  `contentSubtype: "facturae"` para identificarlos en el listado y
  saber que la propuesta será fiable sin ajuste manual.
- **`fromInbox.archive: false`** solo si necesitas mantener el inbox
  activo (por ejemplo, para vincularlo a varios bills). El uso normal
  es archivar.
- **Para integraciones agentic**, los `propose*` son ideales: solo
  requieren `inbox:read`, no `bills:write` ni `payrolls:write`. Un
  agente puede proponer y dejar al humano (o a otro agente con scopes
  superiores) el acto de crear.

## Errores comunes

- `400 ValidationError` — `upload` con formato inválido o que no
  existe.
- `404 Not Found` — `tas_*` no existe en la empresa.
- `409 Conflict` — `propose*` sobre un item con `status: pending` (el
  escaneo aún no ha terminado).
- `409 Conflict` — `proposePayroll` sobre un item cuya extracción no
  es de nómina.

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
| GET | `/{companyId}/inbox` | `getInboxItems` | `inbox:read` | Lista de items en la bandeja de entrada |
| GET | `/{companyId}/inbox/{id}` | `getInboxItem` | `inbox:read` | Detalle de un item de la bandeja |
| POST | `/{companyId}/inbox` | `createInboxItem` | `inbox:write` | Subir un documento a la bandeja de entrada |
| POST | `/{companyId}/inbox/{id}/proposeBill` | `proposeBill` | `inbox:read` | Propuesta de factura de compra desde un item de la bandeja |
| POST | `/{companyId}/inbox/{id}/proposePayroll` | `proposePayroll` | `inbox:read` | Propuesta de nómina desde un item de la bandeja |
| POST | `/{companyId}/inbox/{id}/proposeTicket` | `proposeTicket` | `inbox:read` | Propuesta de ticket de compra desde un item de la bandeja |
| DELETE | `/{companyId}/inbox/{id}` | `deleteInboxItem` | `inbox:write` | Archivar un item de la bandeja |

## Scopes

- **`inbox:read`** — Lectura de la bandeja de entrada de documentos pendientes de procesar.
- **`inbox:write`** — Subida y gestión de documentos en la bandeja de entrada.
