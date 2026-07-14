---
title: Facturas de venta
audience:
  - developers
status: draft
---

# Facturas de venta

Una **factura de venta** (`invoice`) es el documento fiscal que emite
la empresa para cobrar a sus clientes. Es el recurso más cargado del
catálogo: además del flujo CRUD normal toca dos regímenes fiscales
electrónicos (**TicketBAI** en País Vasco y **VeriFactu** en el resto
de España), tiene flujos especiales para **facturas rectificativas**
y **sustitutivas**, y genera asientos contables, PDFs, ficheros
Facturae y envíos por email.

Para no inflar esta página, los temas regulatorios y los flujos
especializados están en guías propias:

- [TicketBAI](../guides/ticketbai.md) — régimen del País Vasco
  (Araba/Bizkaia/Gipuzkoa).
- [VeriFactu](../guides/verifactu.md) — régimen del resto de España,
  con AEAT.
- [Facturas rectificativas](../guides/invoices-rectificativas.md) —
  R1-R5, vínculo con `correctedInvoice`, coordinación con
  TicketBAI/VeriFactu.
- [Facturas sustitutivas (F3)](../guides/invoices-sustitutivas.md) —
  consolidar simplificadas en una completa.

> En los ejemplos, los UUIDs (`inv_…`, `con_…`, `ban_…`, etc.) son
> ilustrativos.

## Subtipos implícitos

FacturaDirecta **no tiene un campo `invoiceType`** en el schema de
factura. El "tipo" emerge de la combinación de varios campos:

| Subtipo | Cómo se identifica |
|---|---|
| **Completa** | `main.contact` informado (cliente identificado con datos fiscales). |
| **Simplificada** | `main.simplified === true` — el destinatario no tiene datos fiscales completos. Típicamente `main.contact` es `null`. |
| **Externa** | `main.external === true` — generada fuera del programa. **No emite a TicketBAI ni VeriFactu**, aunque sí contabiliza. |
| **Sustitutiva** | `main.substitution === true` con `main.substitutedInvoices` rellenado. Se crea con el endpoint dedicado [`POST /invoices/substitution`](#crear-factura-sustitutiva). Corresponde a F3 en VeriFactu. Ver [guía de Sustitutivas](../guides/invoices-sustitutivas.md). |
| **Rectificativa** | Asociada a una factura previa vía `main.correctedInvoice` y/o emitida en una serie de rectificativas (`invoiceType: complete_correction` o `simplified_correction` en [settings](./settings.md)). Ver [guía de Rectificativas](../guides/invoices-rectificativas.md). |

> El campo `main.corrective` existe en el schema pero está marcado
> como **reservado para uso futuro y no debe usarse aún**. Su
> activación está pendiente de una fase posterior.

## Estados

`state` es un campo **calculado** (no se envía en el body). Se
deriva de los flags del documento y del importe pagado vs total:

| Estado | Cuándo |
|---|---|
| `draft` | `main.draft === true`. Provisional, no contabiliza. |
| `pending` | Definitiva, sin cobros suficientes y no vencida. |
| `overdue` | Definitiva, sin cobrar al llegar `main.dueDate`. |
| `paid` | Cobros totales = importe. |
| `overpaid` | Cobros totales > importe. |
| `voided` | `main.voided === true`. Anulada. |

Cada factura tiene un único estado en cada momento: los valores son
excluyentes entre sí. En el filtro `state` se pueden indicar varios y se
devuelven las facturas cuyo estado sea cualquiera de ellos.

Las transiciones no se imponen como máquina de estados: se calculan
en cada lectura desde los pagos y fechas actuales.

## Estructura mínima

`Invoice` tiene tres bloques raíz: `main` (datos editables),
`accounting`/`books` (apuntes y libros, generados) y `meta`
(metadatos internos — incluye los xml y respuestas de TicketBAI y
VeriFactu, ver [TicketBAI](../guides/ticketbai.md) y
[VeriFactu](../guides/verifactu.md)).

Los campos de `main` más importantes para crear una factura:

| Campo | Requerido | Significado |
|---|---|---|
| `contact` | Sí salvo simplificada | ID del cliente (`con_*`). En simplificadas: `null`. |
| `date` | Sí | Fecha de emisión (`YYYY-MM-DD`). |
| `dueDate` | Sí | Fecha de vencimiento. |
| `currency` | Sí | ISO 4217. |
| `exchangeRate` | Sí | Tipo de cambio a la moneda de la empresa. 1 si coincide. |
| `docNumber` | Sí | `{ series: "...", number?: N }`. Si omites `number`, se asigna automáticamente. |
| `fiscalPosition` | Sí | Régimen tributario aplicable. Determina los impuestos disponibles. |
| `lines` | Sí | Array de líneas (ver más abajo). |
| `theme` | Sí | ID de la plantilla de impresión. Obtenible con [`GET /settings/themes`](./settings.md#lista-de-plantillas). |
| `account` | Sí | Cuenta contable de ingresos (típicamente 700*). |
| `draft` | Sí | `true` → provisional; `false` → definitiva (contabiliza). |
| `voided` | Sí | `true` solo para anular. |
| `taxIncludedPrices` | No (default `false`) | Si los precios de las líneas incluyen impuestos. |
| `external` | No | `true` para facturas creadas fuera del programa (no emite TicketBAI/VeriFactu). |
| `paymentMethod` | No | ID de [método de cobro](./payment-methods.md). |
| `emails`, `emailsCc`, `emailsBcc` | No | Destinatarios por defecto para el envío por email (CSV en un solo string). |
| `correctedInvoice` | No | ID de la factura rectificada cuando esta es rectificativa. Ver [Rectificativas](../guides/invoices-rectificativas.md). |
| `verifactu` | No | Parámetros específicos VeriFactu (`TipoFactura`, `DescripcionOperacion`, etc.). Ver [VeriFactu](../guides/verifactu.md). |
| `ticketbai` | No | Parámetros específicos TicketBAI (`causaExencion`, `claveTipoFacturaRectificativa`, etc.). Ver [TicketBAI](../guides/ticketbai.md). |
| `customFields` | No | Map de strings: `{ "miCampo": "valor" }`. |
| `owner` | No | Usuario responsable de la factura. Se usa con roles personalizados para limitar la visibilidad a documentos asignados. |

Si creas la factura con OAuth y no envías `owner`, la API asigna como
responsable al usuario autenticado. Si usas una apiKey, la factura queda
sin responsable salvo que envíes `owner` en el body.

Cada elemento de `lines` (`InvoiceMainLine`):

| Campo | Requerido | Significado |
|---|---|---|
| `account` | Sí | Cuenta contable de la línea (override de `main.account`). |
| `quantity` | Sí | Cantidad. |
| `unitPrice` | Sí | Precio unitario. |
| `discount` | Sí | Descuento absoluto en la moneda de la factura. |
| `discountRate` | Sí | Descuento porcentual (0–100). |
| `lineTotal` | Sí | Total de la línea (`quantity * unitPrice * (1 - discountRate/100) - discount`). |
| `tax` | Sí | Array de IDs de impuestos. Ver [guía de Impuestos](../guides/taxes.md). |
| `text` | Sí | Descripción. |
| `origin` | No | ID del documento de origen (presupuesto, albarán) para trazar procedencia. |
| `document` | No | ID del producto si la línea se genera de catálogo. |
| `facturae`, `verifactu` | No | Sub-campos específicos para Facturae y VeriFactu por línea. |

## Operaciones

- [Lista de facturas](#lista-de-facturas)
- [Crear factura](#crear-factura)
- [Crear factura sustitutiva](#crear-factura-sustitutiva)
- [Obtener una factura](#obtener-una-factura)
- [Actualizar factura](#actualizar-factura)
- [Anular factura](#anular-factura)
- [Borrar factura](#borrar-factura)
- [Actualizar etiquetas](#actualizar-etiquetas)
- [Registrar cobros](#registrar-cobros)
- [Enviar por email](#enviar-por-email)
- [Generar PDF](#generar-pdf)
- [Generar Facturae](#generar-facturae)
- [Adjuntos](#adjuntos)

## Lista de facturas

`GET /{companyId}/invoices` devuelve las facturas de la empresa,
paginadas.

**Parámetros de consulta específicos** (los más comunes; lista
completa en el Swagger UI):

- **`minDate`**, **`maxDate`** — rango por fecha de la factura.
- **`series`** — filtrar por serie de numeración.
- **`contact`** — filtrar por ID de cliente.
- **`state`** — filtrar por estado (`draft`, `pending`, `overdue`,
  `paid`, `overpaid`, `voided`). Cada factura solo tiene uno; si se
  indican varios, el filtro incluye cualquiera de ellos.
- **`draft`** — por defecto solo se incluyen facturas definitivas;
  `only` devuelve solo borradores y `all` mezcla definitivas y borradores.
  Los borradores no generan asientos contables, por lo que al incluirlos
  el recuento puede no coincidir con el diario.
- **`simplified`** — `true` para solo simplificadas.
- **`external`** — `true` para solo externas.
- **`voided`** — `true` para solo anuladas.
- **`sortBy`** — admite `date`, `series`, `formattedSeries`, `number`,
  `total`, `currency`, `country`, `creationDate` y `modificationDate`.
  Repite el parámetro para combinar criterios y antepone `-` para orden
  descendente (por ejemplo, `sortBy=-date&sortBy=number`).
- **Tags**: `allTheseTags`, `anyOfTheseTags`, `hasTags`.

**Parámetros globales aceptados:**

Acepta además los parámetros estándar `offset`, `limit`,
`minCreationDate`, `maxCreationDate`, `minModificationDate`,
`maxModificationDate` y el header `accept-version`. Ver
[Paginación](../guides/pagination.md).

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/invoices?minDate=2026-01-01&state=pending&limit=50"
```

## Crear factura

`POST /{companyId}/invoices` crea una factura nueva.

**Body:** `{ content: Invoice, tags?: string[] }`. El `content`
contiene todo el cuerpo de la factura (típicamente solo `main`; los
campos calculados se generan en el servidor).

**Reglas de negocio relevantes:**

- Si **omites `docNumber.number`**, se asigna automáticamente
  basándose en el último número emitido para esa serie ese año. La
  serie controla además la sustitución de `##`/`####` por el año
  (ver [settings](./settings.md#lista-de-series-de-numeración)).
- Si **omites `draft`**, se aplica el valor por defecto de la
  empresa (configuración).
- Si la empresa tiene TicketBAI o VeriFactu activos y la factura
  encaja en su ámbito (no es `external`), el envío al organismo
  fiscal se prepara en el momento del guardado. Ver
  [TicketBAI](../guides/ticketbai.md) y
  [VeriFactu](../guides/verifactu.md).
- `main.simplified` es **obligatorio** en el schema; envíalo
  explícitamente. Si `main.contact` es `null`, debe ser `true`.

**Parámetros globales aceptados:** `accept-version`.

###### Ejemplo de request JSON (factura completa, regalo a un cliente)

```json
{
  "content": {
    "main": {
      "contact": "con_2e3f4a18-9b7c-4d6a-8e1f-5c2d3b4a7e9f",
      "date": "2026-05-13",
      "dueDate": "2026-06-12",
      "currency": "EUR",
      "exchangeRate": 1,
      "docNumber": { "series": "F-##-" },
      "fiscalPosition": "general",
      "account": "700000",
      "theme": "default",
      "draft": false,
      "voided": false,
      "simplified": false,
      "lines": [
        {
          "account": "700000",
          "quantity": 1,
          "unitPrice": 100,
          "discount": 0,
          "discountRate": 0,
          "lineTotal": 100,
          "tax": ["S_IVA_21"],
          "text": "Servicios de consultoría"
        }
      ]
    }
  },
  "tags": ["consultoria"]
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d @invoice.json \
  "https://app.facturadirecta.com/api/$COMPANY_ID/invoices"
```

## Crear factura sustitutiva

`POST /{companyId}/invoices/substitution` crea una **factura
completa que sustituye a un conjunto de facturas simplificadas**
(F3 en VeriFactu). Es un endpoint dedicado porque la construcción
del cuerpo (líneas, separadores, totales) la hace el servidor.

**Body:**

- **`contact`** (obligatorio) — ID del cliente al que se le emite la
  factura completa.
- **`substitutedInvoices`** (obligatorio) — array no vacío de UUIDs
  de facturas simplificadas a sustituir. Sin duplicados.
- **`date`**, **`docNumber`**, **`notes`**, **`internalNotes`**,
  **`tags`** — opcionales.

**Validaciones:**

- Todas las facturas referenciadas deben existir y **ser
  simplificadas** (`main.simplified === true`).
- **Ninguna** puede ser externa (`main.external === true`).
- **Misma `currency`** en todas.
- **Mismo `taxIncludedPrices`** en todas.
- Si alguna tiene `main.contact`, todas las que lo tengan deben
  compartir el mismo `con_*`.
- Ninguna puede estar ya sustituida en otra F3.

Detalles del flujo y ejemplos: ver
[guía de Facturas sustitutivas](../guides/invoices-sustitutivas.md).

**Parámetros globales aceptados:** `accept-version`.

## Obtener una factura

`GET /{companyId}/invoices/{id}` devuelve una factura por ID.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/invoices/inv_4a9c2e58-3b1f-4d6a-8e2c-7f1a9b3d5c4e"
```

## Actualizar factura

`PUT /{companyId}/invoices/{id}` reemplaza el `content` de la
factura.

**Reglas relevantes:**

- `docNumber` **inmutable** una vez asignado.
- Pasar `main.voided=true` **anula** la factura (ver
  [Anular](#anular-factura)).
- Si la factura tiene metadatos de TicketBAI/VeriFactu ya enviados,
  algunos cambios disparan reemisión o anulación + nueva emisión.
  Detalles en las respectivas guías.

**Parámetros globales aceptados:** `accept-version`.

## Anular factura

**No existe un endpoint `/void` dedicado.** La anulación se hace con
`PUT /{companyId}/invoices/{id}` enviando `main.voided = true`.

**Efectos:**

- El estado pasa a `voided`.
- Se generan asientos contables de **reversión**.
- Si la factura está en TicketBAI o VeriFactu, se prepara el envío
  de anulación al organismo (`meta.ticketbai.voided` /
  `meta.verifactu.registroAnulacion`).
- Se emite el webhook **`invoice.voided`**.

Una factura **anulada no es lo mismo que una factura borrada**: la
anulada queda en el sistema con estado `voided`; la borrada se
archiva y desaparece de los listados (ver siguiente sección).

## Borrar factura

`DELETE /{companyId}/invoices/{id}` archiva la factura.

**Restricciones:**

- Solo se pueden borrar facturas **no contabilizadas** (típicamente
  en `draft`). Si la factura ya generó asientos contables, el borrado
  fallará: usa **anulación** en su lugar.
- El borrado es **archivado lógico** (`archived = true`); la factura
  no se elimina físicamente y desaparece de los listados.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/invoices/inv_4a9c2e58-3b1f-4d6a-8e2c-7f1a9b3d5c4e"
```

## Actualizar etiquetas

`PUT /{companyId}/invoices/{id}/tags` reemplaza el array de tags de
la factura sin tocar el `content`. Body: `{ tags: string[] }`. Ver
los recursos análogos ([bills](./bills.md), [estimates](./estimates.md))
para el patrón común de tags.

## Registrar cobros

`POST /{companyId}/invoices/{id}/payments` registra uno o varios
cobros contra la factura.

**Body:**

```json
{
  "payments": [
    { "date": "2026-05-13", "bank": "ban_…", "amount": 121 }
  ]
}
```

**Reglas:**

- **`bank`** (ID `ban_*`) es obligatorio en cada pago.
- **`amount`** es opcional: si lo omites, se cobra el saldo
  pendiente completo. Solo se admite **un pago sin `amount`** y debe
  ir en la **última** posición del array.
- Genera asientos contables automáticamente (banco vs cliente).
- Si los pagos cierran el saldo, el estado pasa a `paid`. Si lo
  exceden, pasa a `overpaid`.
- Emite el webhook **`invoice.updated`** (no hay evento específico de
  cobro).

**Parámetros globales aceptados:** `accept-version`.

## Enviar por email

`PUT /{companyId}/invoices/{id}/send` envía la factura por email
desde la cuenta SMTP configurada en la empresa.

**Body** (todos opcionales):

- **`to`**, **`cc`**, **`bcc`** — CSV de destinatarios. Si los
  omites, se usan `main.emails`, `main.emailsCc`, `main.emailsBcc`
  del documento.
- **`subject`** — asunto custom; en su defecto se aplica el
  configurado por la empresa.
- **`html`** — cuerpo HTML del mensaje.
- **`sendFacturae`** — `true` para adjuntar también el XML Facturae
  (la factura debe ser elegible: cliente con `taxCode`, etc.).
- **`continueWithoutFacturae`** — `true` para no fallar si el
  Facturae no puede generarse (sin él, solo se envía el PDF).

**Efectos:**

- Adjunta automáticamente el PDF de la factura.
- Adjunta el Facturae XML si se solicita y es elegible.
- Emite el webhook **`invoice.sent`** al completarse.

**Parámetros globales aceptados:** `accept-version`.

## Generar PDF

`PUT /{companyId}/invoices/{id}/pdf` genera el PDF de la factura
aplicando la plantilla `main.theme`.

**Respuesta:** `{ url, filename }`. La URL es temporal (~24h);
vuelve a llamar al endpoint para obtener una nueva si caduca.

**Parámetros globales aceptados:** `accept-version`.

## Generar Facturae

`PUT /{companyId}/invoices/{id}/facturae` genera el XML Facturae
**versión 3.2.1** (formato estándar B2B español).

**Respuesta:** `{ url, filename }` con la URL del XML.

**Restricciones:**

- La factura debe ser elegible (cliente con `taxCode`, dirección
  fiscal completa, etc.).
- Para facturas rectificativas, los campos
  `Corrective_*` se generan automáticamente desde
  `main.correctedInvoice` y los campos VeriFactu/TicketBAI.

**Parámetros globales aceptados:** `accept-version`.

## Adjuntos

Patrón estándar compartido con el resto de documentos:

- **`POST /{companyId}/invoices/{id}/attachments`** — body
  `{ uploadIds: string[] }`. Adjunta archivos previamente subidos
  con [Uploads](./uploads.md).
- **`GET /{companyId}/invoices/{id}/attachments`** — lista los
  adjuntos.
- **`DELETE /{companyId}/invoices/{id}/attachments/{attachmentIndex}`** —
  borra por **índice** (0-based) en el array.

**Límite:** **10 adjuntos por documento**. El POST con un total
superior devuelve `400`.

## Webhooks

Eventos que emite el recurso:

- **`invoice.created`** — alta de factura.
- **`invoice.updated`** — modificación del `content`, incluido el
  registro de pagos.
- **`invoice.archived`** — borrado (archivado lógico).
- **`invoice.unarchived`** — restauración (no expuesta hoy por API
  pública; reservada para procesos internos).
- **`invoice.sent`** — envío por email completado.
- **`invoice.voided`** — anulación.
- **`invoice.verifactu_sent`** — envío exitoso a AEAT confirmado.
  Ver [VeriFactu](../guides/verifactu.md).

> No existen eventos `invoice.ticketbai_*`. Para TicketBAI, el
> estado se consulta en `meta.ticketbai` del documento. Ver
> [TicketBAI](../guides/ticketbai.md).

Detalle de la entrega (HMAC, reintentos, formato) en
[Webhooks](./webhooks.md).

## Recomendaciones

- **Tres lecturas obligadas antes de tu integración**: la
  [guía de TicketBAI](../guides/ticketbai.md), la
  [guía de VeriFactu](../guides/verifactu.md) y la
  [guía de Impuestos](../guides/taxes.md). Conocer cuándo y cómo se
  emite electrónicamente la factura evita la mayoría de los errores
  reportados.
- **Para crear, deja que el servidor asigne el número**: no envíes
  `docNumber.number` salvo que tengas un caso de uso muy concreto
  (importaciones desde otro sistema). Así evitas colisiones y
  errores de huecos en la numeración.
- **Para integraciones que envían a clientes finales**, registra los
  webhooks `invoice.created`, `invoice.voided`, `invoice.sent` y
  `invoice.verifactu_sent`. Son los que disparan acciones externas
  (notificaciones, replicación a CRMs, descargas).
- **Multiprovincias**: si tu integración opera con empresas en
  País Vasco y en el resto de España, no asumas el mismo flujo. Lee
  ambas guías regulatorias.
- **Factura externa**: úsala con cuidado. `main.external = true`
  marca la factura como creada fuera del programa. Contabiliza
  normalmente, pero **no la envía a TicketBAI ni VeriFactu**. Es el
  modo apropiado para registrar facturas históricas o las emitidas
  con otro software fiscal, pero **no** para evitar la obligación
  electrónica.
- **Rectificativas y sustitutivas**: cada una tiene su flujo
  específico. Las rectificativas se crean con el CRUD normal
  marcando `correctedInvoice` y la serie apropiada; las sustitutivas
  tienen un endpoint dedicado. Lee las guías antes de implementar.

## Errores comunes

- `400 ValidationError` — combinaciones incoherentes:
  - `main.contact = null` con `main.simplified = false`.
  - `main.exchangeRate <= 0` o ausente cuando `currency ≠
    settings.currency`.
  - `main.docNumber.series` no presente en
    `settings.numbering.invoice`.
  - `line.tax` con un `TaxId` que no está en el catálogo de
    impuestos de venta de la empresa.
- `400` en `POST /payments` — el array contiene más de un pago sin
  `amount`, o el pago sin `amount` no está en la última posición.
- `400` en `POST /invoices/substitution` — alguna restricción no se
  cumple (factura no simplificada, monedas distintas, ya
  sustituida...).
- `404 Not Found` — factura inexistente o archivada.
- `409 Conflict` — intento de borrar una factura ya contabilizada.
  Anula en su lugar.
- `413 Payload Too Large` — adjuntos por encima del límite (10).

Ver [Errores y validaciones](../guides/errors.md) para el formato
general.

## Referencia exhaustiva

Esta página resume el contrato y los matices funcionales del
recurso. Para el contrato completo del schema (todos los campos del
`Invoice`, los `$ref` a `documentCounterpart`, `invoiceMainFacturae`,
`invoiceMainVerifactu`, `invoiceMetaTicketbai`,
`invoiceMetaVerifactu`, etc.), consulta el
[Swagger UI](https://www.facturadirecta.com/api) o el
[openapi crudo](https://app.facturadirecta.com/openapi.json).

Los detalles regulatorios y los flujos especiales viven en las
guías dedicadas:

- [TicketBAI](../guides/ticketbai.md)
- [VeriFactu](../guides/verifactu.md)
- [Rectificativas](../guides/invoices-rectificativas.md)
- [Sustitutivas (F3)](../guides/invoices-sustitutivas.md)

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| GET | `/{companyId}/invoices` | `getInvoices` | `invoices:read` | Lista de facturas |
| GET | `/{companyId}/invoices/{id}` | `getInvoice` | `invoices:read` | Obtener una factura |
| GET | `/{companyId}/invoices/{id}/attachments` | `getInvoiceAttachments` | `invoices:read` | Listar adjuntos de una factura de venta |
| POST | `/{companyId}/invoices` | `createInvoice` | `invoices:write` | Crear factura |
| POST | `/{companyId}/invoices/{id}/attachments` | `addInvoiceAttachments` | `invoices:write` | Vincular adjuntos a una factura |
| POST | `/{companyId}/invoices/{id}/payments` | `createInvoicePayments` | `invoices:write` | Crear pagos para una factura |
| POST | `/{companyId}/invoices/substitution` | `createSubstitutionInvoice` | `invoices:write` | Crear factura sustitutiva de simplificadas |
| PUT | `/{companyId}/invoices/{id}` | `updateInvoice` | `invoices:write` | Actualizar factura |
| PUT | `/{companyId}/invoices/{id}/facturae` | `createInvoiceFacturae` | `invoices:read` | Generar la factura en formato facturae |
| PUT | `/{companyId}/invoices/{id}/pdf` | `createInvoicePDF` | `invoices:read` | Generar la factura en formato PDF |
| PUT | `/{companyId}/invoices/{id}/send` | `sendInvoice` | `invoices:write` | Enviar factura por correo electrónico |
| PUT | `/{companyId}/invoices/{id}/tags` | `updateInvoiceTags` | `invoices:write` | Actualizar etiquetas de factura |
| DELETE | `/{companyId}/invoices/{id}` | `deleteInvoice` | `invoices:write` | Borrar factura |
| DELETE | `/{companyId}/invoices/{id}/attachments/{attachmentIndex}` | `deleteInvoiceAttachment` | `invoices:write` | Eliminar adjunto de una factura de venta |

## Scopes

- **`invoices:read`** — Lectura de facturas.
- **`invoices:write`** — Modificación de facturas.
