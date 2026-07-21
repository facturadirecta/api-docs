---
title: Facturas de compra y tickets
audience:
  - developers
status: draft
---

# Facturas de compra y tickets

Una factura de compra representa un gasto: una factura que has recibido de
un proveedor, o un ticket de caja (con o sin identificación fiscal del
proveedor). En FacturaDirecta ambos casos se modelan con el mismo recurso
(`bill`), distinguidos por un campo `subtype`.

**Flujos para crear facturas de compra.** Existen dos caminos:

- **Construir el body manualmente** y enviarlo a [`POST /bills`](#crear-factura-de-compra).
  Es el flujo descrito en esta página.
- **Desde la bandeja de entrada**: subes el PDF (o Facturae) a la
  bandeja, el sistema extrae los datos por escaneo automático, llamas
  a [`POST /inbox/{id}/proposeBill`](./inbox.md#proponer-factura-de-compra)
  para obtener un prototipo prerellenado, lo revisas y lo envías a
  `POST /bills` con `fromInbox: { taskId }` (ver
  [parámetro `fromInbox`](#crear-factura-de-compra) más abajo). Este
  flujo vincula la factura con su origen en la bandeja y reutiliza el
  PDF sin re-upload.

## Factura de compra vs ticket

Por defecto, `content.main.subtype` no se envía y el documento es una
**factura de compra** convencional. Para registrar un ticket de caja,
envía `content.main.subtype: "ticket"`. La diferencia entre ambos:

- **Factura de compra** — proveedor con identificación fiscal, número de
  factura del proveedor (`referenceNumber`), tratamiento fiscal estándar.
- **Ticket** — gasto registrado típicamente desde un escaneo o entrada
  manual rápida. Suele venir sin contacto identificado y sin número de
  factura del proveedor.

Ambos se listan, se filtran y se gestionan con los mismos endpoints. El
`subtype` solo cambia la categorización a efectos de informes y plantillas.

## Estructura

Forma común a los documentos de gasto de FacturaDirecta:

- `content.type` — siempre `"bill"`.
- `content.uuid` — identificador inmutable.
- `content.main` — datos del documento (contacto, fechas, divisa, líneas,
  totales, modo de cálculo, fiscalidad, pagos a plazos, ...).
- `content.attachments` — adjuntos vinculados (ver [Adjuntos](#adjuntos)).
- `content.meta` — metadatos internos.

En respuestas, además, en el nivel raíz: `tags`, `creationDate`,
`modificationDate`, `related`.

### Líneas de detalle

`content.main.lines` es un array. Cada línea requiere `text`, `quantity` y
`unitPrice`. Otros campos relevantes:

- `discount` o `discountRate` — descuento absoluto o porcentual.
- `tax` — array de **identificadores de impuesto** (`TaxId`) que aplican
  a la línea. Los IDs se obtienen del catálogo de **compras** de la
  empresa con `GET /{companyId}/settings/taxes/purchases`. Pueden
  aplicarse varios impuestos a una misma línea, típicamente IVA +
  retención de IRPF. **Los impuestos autorepercutidos** (intracomunitarios,
  inversión del sujeto pasivo, importación de servicios) **no se envían
  aquí**: el sistema los añade automáticamente al agregado del documento
  a partir de su impuesto base. Ver [Impuestos](../guides/taxes.md) para
  el modelo completo.

> En los ejemplos de esta página:
>
> - Los **UUIDs** (`con_2c4f8e21-…`, `bil_4a9c2e58-…`, `ban_5e7d8a31-…`,
>   `upl_6f8e9b42-…`) son ilustrativos. Cada empresa tiene los suyos;
>   sustitúyelos por los identificadores reales que devuelve la API.
> - El **ID de impuesto** `P_IVA_21_SV` es el del catálogo por defecto
>   para IVA 21% en compra de **servicios**. Para bienes corrientes el
>   ID típico es `P_IVA_21_BC`. El sufijo identifica el régimen
>   aplicable (servicios, bienes corrientes, bienes de inversión).
>   Recupera el catálogo real con `GET /{companyId}/settings/taxes/purchases`.
- `lineTotal` — total de la línea; la API lo calcula si lo omites.
- `account` — cuenta contable de la línea; por defecto se toma de la
  cuenta de gastos por defecto del proveedor o de la empresa.

### Modos de cálculo de totales: `totalProvided` vs `calcTotal`

`content.main.mode` controla cómo se interpretan los totales del
documento:

- **`calcTotal`** (modo por defecto) — los totales (`total`,
  `totalBeforeTaxes`, `linesTotal`, `taxes`) se **calculan** a partir de
  las líneas. Conviene dejarlos vacíos al crear y leerlos de la respuesta.
- **`totalProvided`** — los totales del cuerpo se **toman como dados**;
  no se recalculan a partir de las líneas. Útil cuando importas una
  factura escaneada que solo declara totales agregados, o cuando hay
  redondeos del proveedor que quieres respetar.

`taxIncludedPrices: true` indica que `unitPrice` ya incluye IVA. Por
defecto es `false`.

### Vencimientos múltiples

Para dividir el pago de una factura de compra, envía `main.installments` al
crearla o actualizarla. Cada elemento requiere:

- `date` — fecha del vencimiento en formato `YYYY-MM-DD`.
- `amount` — importe previsto para ese vencimiento.
- `currency` — moneda del importe en formato ISO 4217.

Por ejemplo, una factura de compra de 121 EUR puede dividirse así:

```json
{
  "dueDate": "2026-08-15",
  "installments": [
    { "date": "2026-07-15", "amount": 60.5, "currency": "EUR" },
    { "date": "2026-08-15", "amount": 60.5, "currency": "EUR" }
  ]
}
```

Incluye estos campos dentro de `content.main` en `POST /{companyId}/bills` o
`PUT /{companyId}/bills/{billId}`. El servidor ajusta el importe del último
elemento para que la suma coincida con el total calculado de la factura y
establece `dueDate` en la fecha más tardía. Con un método de pago por
transferencia SEPA, cada vencimiento pendiente se puede seleccionar como un
pago separado al preparar una remesa. Si no necesitas dividir el pago,
`dueDate` basta para representar un único vencimiento.

## Estados

`state` es un campo calculado a partir de los flags del documento, sus
pagos y sus vencimientos:

| Estado | Cuándo |
|---|---|
| `draft` | `main.draft === true`. Provisional; no genera asientos contables. |
| `pending` | Definitiva, con saldo pendiente que aún no ha vencido. |
| `overdue` | Definitiva, con saldo pendiente vencido según sus vencimientos. |
| `paid` | Saldo pendiente cero. |
| `overpaid` | Los pagos superan el importe esperado. |
| `voided` | `main.voided === true`. Anulada. |

Cada factura de compra o ticket tiene un único estado en cada momento:
los valores son excluyentes entre sí. En el filtro `state` se pueden
indicar varios y se devuelven los documentos cuyo estado sea cualquiera
de ellos.

Al crear o actualizar, `draft: true` guarda el documento como provisional.
Si se omite, se aplica la configuración por defecto de la empresa. Los
borradores no aparecen en el diario, por lo que un listado que los incluya
puede tener un recuento distinto al de los movimientos contables.

Con `voided: true` la factura queda anulada. Solo se puede anular en
empresas que soportan el registro de facturas de compra en sistemas
externos (TicketBAI, VeriFactu...). En otros casos, usa el borrado.

### Fiscalidad y casos especiales

Bills tiene varios campos para tratamiento fiscal específico que esta
guía menciona pero no desarrolla, porque cada uno merece su propia guía:

- **`ticketbai`** — datos de cumplimiento TicketBAI para empresas del
  País Vasco.
- **`correctedBill`** — ID de la factura de compra que esta factura
  rectifica.
- **`referenceNumber`** — número de factura asignado por el proveedor.
  No confundir con `docNumber` (numeración interna).
- **`transactionDate`** — fecha de la transacción cuando difiere de la
  fecha del documento.
- **`depreciable`** y **`depreciationSettings`** — amortización contable
  para bienes inventariables.
- **`installments`** — calendario de pagos a plazos. Ver
  [Vencimientos múltiples](#vencimientos-múltiples).
- **`manualAccounting`** y **`manualTotals`** — overrides contables
  manuales para casos donde la lógica automática no aplica.
- **`nonDeductibleInIRPF`** — gasto no deducible en IRPF.

## Operaciones

- [Lista de facturas de compra](#lista-de-facturas-de-compra)
- [Crear factura de compra](#crear-factura-de-compra)
- [Obtener una factura de compra](#obtener-una-factura-de-compra)
- [Actualizar factura de compra](#actualizar-factura-de-compra)
- [Borrar factura de compra](#borrar-factura-de-compra)
- [Actualizar etiquetas](#actualizar-etiquetas)
- [Registrar pagos](#registrar-pagos)
- [Adjuntos](#adjuntos)

## Lista de facturas de compra

`GET /{companyId}/bills` devuelve las facturas de compra y tickets de la
empresa, paginados.

**Parámetros de consulta principales:**

- `minDate`, `maxDate` — rango por fecha del documento (`YYYY-MM-DD`).
- `contact` — ID de contacto.
- `hasContact` — `true`/`false` para filtrar con o sin contacto (los
  tickets suelen ir sin contacto).
- `minTotal`, `maxTotal` — rango de importe.
- `currency` — código ISO 4217.
- `country` — código país ISO 3166-1 Alpha-2.
- `state` — estado del documento (`draft`, `pending`, `overdue`, `paid`,
  `overpaid`, `voided`). Cada documento solo tiene uno; si se indican
  varios, el filtro incluye cualquiera de ellos.
- `draft` — por defecto solo se incluyen documentos definitivos; `only`
  devuelve solo borradores y `all` mezcla definitivos y borradores. Los
  borradores no generan asientos contables, por lo que al incluirlos el
  recuento puede no coincidir con el diario.
- `allTheseTags`, `anyOfTheseTags`, `hasTags`.
- `sortBy` — admite `date`, `total`, `currency`, `country`,
  `creationDate` y `modificationDate`. Repite el parámetro para combinar
  criterios y antepone `-` para orden descendente.
- `related` — recursos a expandir.

Ver [Paginación](../guides/pagination.md) para los parámetros comunes.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/bills?minDate=2026-01-01&state=pending&limit=50"
```

## Crear factura de compra

`POST /{companyId}/bills` crea una factura de compra o ticket.

**Parámetros del body:**

- `content.type` — siempre `"bill"`.
- `content.main` — campos del documento (al menos `contact` o
  `subtype: "ticket"` y las `lines` con `text`, `quantity`, `unitPrice`).
- `tags` (opcional).
- `fromInbox` (opcional) — vincula la factura a un item de la
  [bandeja de entrada](./inbox.md). Estructura:
  - `taskId` (obligatorio dentro de `fromInbox`) — el `tas_*` del item
    de la bandeja.
  - `archive` (opcional, default `true`) — si `true` (por defecto), el
    inbox item se archiva tras crear la factura.

  El adjunto del inbox se **reutiliza** como adjunto de esta factura
  (sin re-upload del binario S3) y queda enlazada al inbox vía
  `content.main.task`. Es el flujo recomendado tras
  [`POST /inbox/{id}/proposeBill`](./inbox.md#proponer-factura-de-compra):
  el endpoint `propose*` devuelve el `content` propuesto, tú lo ajustas
  si quieres y lo envías aquí con `fromInbox: { taskId }`.

**Notas:**

- Si `contact` apunta a un ID existente con faceta de proveedor
  (`accounts.provider` configurada), sus datos fiscales se toman de él.
- Para registrar un ticket de caja sin proveedor identificado, omite
  `contact` y envía `subtype: "ticket"`.
- Por defecto el modo es `calcTotal` (los totales se calculan). Para
  importar un documento con totales fijos del proveedor, usa
  `mode: "totalProvided"`.

###### Ejemplo de request JSON

```json
{
  "content": {
    "type": "bill",
    "main": {
      "contact": "con_2c4f8e21-7b3a-4d9c-9e1f-a8b7c6d5e4f3",
      "date": "2026-04-15",
      "referenceNumber": "F-2026-00123",
      "currency": "EUR",
      "lines": [
        {
          "text": "Servicios de hosting Q2 2026",
          "quantity": 1,
          "unitPrice": 240.00,
          "tax": ["P_IVA_21_SV"]
        }
      ]
    }
  },
  "tags": ["hosting"]
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '@bill.json' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/bills"
```

Crear desde un item de la bandeja de entrada:

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{
    "content": <prototipo devuelto por proposeBill>,
    "fromInbox": { "taskId": "tas_3a7f9c12-2d4e-4b8a-9c1f-5d6e8f0a3b2c" }
  }' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/bills"
```

## Obtener una factura de compra

`GET /{companyId}/bills/{id}` devuelve una factura de compra por ID.

**Parámetros:**

- `related` (opcional) — recursos relacionados a expandir, por ejemplo
  `payments` o `attachments`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/bills/bil_4a9c2e58-3b1f-4d6a-8e2c-7f1a9b3d5c4e?related=payments"
```

## Actualizar factura de compra

`PUT /{companyId}/bills/{id}` sustituye **el contenido completo** de la
factura de compra. No es un PATCH: lo que no envíes, se borra.

**Restricciones:**

- Una factura de compra anulada (`voided: true`) no se puede modificar.
- Cambios en facturas de empresas con TicketBAI/VeriFactu activo pueden
  desencadenar la emisión de eventos de cumplimiento.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '@bill.json' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/bills/bil_4a9c2e58-3b1f-4d6a-8e2c-7f1a9b3d5c4e"
```

## Borrar factura de compra

`DELETE /{companyId}/bills/{id}` elimina una factura de compra.

**Restricciones:**

- Si la factura tiene pagos asociados, se rechaza con `409 Conflict`.
  Elimina primero los pagos.
- En empresas con TicketBAI/VeriFactu sobre compras, conviene usar
  `voided: true` en vez de borrado para mantener el rastro fiscal.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/bills/bil_4a9c2e58-3b1f-4d6a-8e2c-7f1a9b3d5c4e"
```

## Actualizar etiquetas

`PUT /{companyId}/bills/{id}/tags` reemplaza el conjunto de etiquetas sin
tocar el resto del contenido.

###### Ejemplo de request JSON

```json
{
  "tags": ["hosting", "q2-2026"]
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '{"tags":["hosting","q2-2026"]}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/bills/bil_4a9c2e58-3b1f-4d6a-8e2c-7f1a9b3d5c4e/tags"
```

## Registrar pagos

`POST /{companyId}/bills/{id}/payments` registra uno o varios pagos contra
una factura de compra. El body es un **array** de pagos.

**Parámetros de cada pago:**

- `bank` (obligatorio) — ID del banco/cuenta de tesorería donde se anota
  el pago.
- `amount` (opcional) — importe del pago en la moneda del documento. Si
  se omite, salda **todo el pendiente** de la factura. Solo un pago del
  array puede omitir `amount`, y debe ser el **último** del array.
- `date` (opcional) — fecha en la que contabilizar el pago. Por defecto,
  la fecha actual.
- `companyCurrencyAmount` (opcional) — importe a contabilizar en la
  moneda de la contabilidad. Solo aplica cuando la moneda de la factura
  difiere de la moneda contable; si no se envía, se calcula con tipo de
  cambio automático según `date`.
- `description` (opcional) — descripción visible en Bancos → Movimientos.
  Sustituye al título por defecto.
- `notes` (opcional) — notas adicionales del pago.

**Respuesta:** la factura completa actualizada (con los pagos registrados
visibles en `related.payments` cuando se consulta).

###### Ejemplo de request JSON

Pago único que salda toda la factura:

```json
[
  { "bank": "ban_5e7d8a31-9c4b-4f6e-a1d3-2b5c7e9f1a4d" }
]
```

Dos pagos parciales seguidos del saldo final:

```json
[
  { "bank": "ban_5e7d8a31-9c4b-4f6e-a1d3-2b5c7e9f1a4d", "amount": 80.00, "date": "2026-04-20" },
  { "bank": "ban_5e7d8a31-9c4b-4f6e-a1d3-2b5c7e9f1a4d", "amount": 80.00, "date": "2026-05-05" },
  { "bank": "ban_5e7d8a31-9c4b-4f6e-a1d3-2b5c7e9f1a4d" }
]
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '[{"bank":"ban_5e7d8a31-9c4b-4f6e-a1d3-2b5c7e9f1a4d"}]' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/bills/bil_4a9c2e58-3b1f-4d6a-8e2c-7f1a9b3d5c4e/payments"
```

## Adjuntos

Una factura de compra puede tener varios adjuntos vinculados (típicamente
el PDF original del proveedor). Los adjuntos **no se suben en el body de
la factura**: primero se crean con `POST /uploads` y después se vinculan
con `POST /bills/{id}/attachments` enviando los `uploadIds`.

### Listar adjuntos

`GET /{companyId}/bills/{id}/attachments` devuelve los adjuntos vinculados.

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/bills/bil_4a9c2e58-3b1f-4d6a-8e2c-7f1a9b3d5c4e/attachments"
```

### Vincular adjuntos

`POST /{companyId}/bills/{id}/attachments` añade adjuntos a partir de
uploads previos.

**Parámetros del body:**

- `uploadIds` — array de identificadores `upl_<uuid>`. Mínimo uno.

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"uploadIds":["upl_6f8e9b42-1d3c-4a7f-b2e4-9c1d5e7f3a6b","upl_7a9b0c53-2e4d-4b8a-a3f5-0d2e6f8a4b7c"]}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/bills/bil_4a9c2e58-3b1f-4d6a-8e2c-7f1a9b3d5c4e/attachments"
```

### Eliminar un adjunto

`DELETE /{companyId}/bills/{id}/attachments/{attachmentIndex}` elimina un
adjunto. `attachmentIndex` es la posición en el array (basada en cero).

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/bills/bil_4a9c2e58-3b1f-4d6a-8e2c-7f1a9b3d5c4e/attachments/0"
```

## Errores comunes

- `400 ValidationError` — falta `subtype: "ticket"` o `contact` cuando
  el documento no es un ticket.
- `400 ValidationError` — `lines` con `quantity` o `unitPrice` inválidos.
- `400 ValidationError` — `mode: "totalProvided"` sin enviar `total`.
- `409 Conflict` — borrado de una factura con pagos asociados.
- `409 Conflict` — modificación de una factura anulada (`voided: true`).
- `409 Conflict` — `voided: true` en una empresa sin registro externo de
  facturas de compra.

Ver [Errores y validaciones](../guides/errors.md) para el formato general
de respuestas de error.

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| GET | `/{companyId}/bills` | `getBills` | `bills:read` | Lista de facturas de compra o tickets |
| GET | `/{companyId}/bills/{id}` | `getBill` | `bills:read` | Obtener una factura de compra o ticket |
| GET | `/{companyId}/bills/{id}/attachments` | `getBillAttachments` | `bills:read` | Listar adjuntos de una factura de compra o ticket |
| POST | `/{companyId}/bills` | `createBill` | `bills:write` | Crear factura de compra o ticket |
| POST | `/{companyId}/bills/{id}/attachments` | `addBillAttachments` | `bills:write` | Vincular adjuntos a una factura de compra o ticket |
| POST | `/{companyId}/bills/{id}/payments` | `createBillPayments` | `bills:write` | Crear pagos para una factura de compra o ticket |
| PUT | `/{companyId}/bills/{id}` | `updateBill` | `bills:write` | Actualizar factura de compra o ticket |
| PUT | `/{companyId}/bills/{id}/tags` | `updateBillTags` | `bills:write` | Actualizar etiquetas de factura de compra o ticket |
| DELETE | `/{companyId}/bills/{id}` | `deleteBill` | `bills:write` | Borrar factura de compra o ticket |
| DELETE | `/{companyId}/bills/{id}/attachments/{attachmentIndex}` | `deleteBillAttachment` | `bills:write` | Eliminar adjunto de una factura de compra o ticket |

## Scopes

- **`bills:read`** — Lectura de facturas de compra y tickets.
- **`bills:write`** — Modificación de facturas de compra y tickets.
