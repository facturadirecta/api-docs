---
title: Albaranes
audience:
  - developers
status: draft
---

# Albaranes

Un albarán documenta la **entrega** de bienes o servicios a un cliente.
Es el justificante de que el suministro se ha realizado, normalmente
antes de emitir la factura correspondiente. Tiene líneas de detalle,
estados de entrega y un ciclo de vida propio.

> En los ejemplos de esta página:
>
> - Los **UUIDs** (`con_2c4f8e21-…`, `den_3a7b9c14-…`, `upl_6f8e9b42-…`)
>   son ilustrativos. Cada empresa tiene los suyos; sustitúyelos por
>   los identificadores reales que devuelve la API.
> - El **ID de impuesto** `S_IVA_21` es el del catálogo por defecto
>   para IVA 21% en ventas. Recupera el catálogo real con
>   `GET /{companyId}/settings/taxes/sales`. Ver
>   [Impuestos](../guides/taxes.md).

## Relación con la factura

La API pública **no expone una operación de conversión** albarán → factura.
La conversión, cuando aplica, se hace desde la interfaz. Si necesitas
emitir una factura a partir de los datos de uno o varios albaranes,
crea la factura directamente con `POST /invoices` con las líneas que
correspondan.

Un albarán convertido o asociado a una factura mantiene su estado en
`baseState`, pero puede aparecer en su `related` con el documento de
venta resultante cuando se consulta.

## Estados

`content.main.baseState` controla el estado declarado del albarán:

- `pending` — emitido pero pendiente de entrega.
- `delivered` — entregado al cliente.
- `closed` — cerrado sin entregar (caducado, cancelado o anulado).
- `rejected` — el cliente ha rechazado la entrega.

Campos que se rellenan típicamente al cambiar `baseState`:

- `deliveredDate`, `deliveredTo` — fecha de entrega y persona del
  cliente que la ha recibido (firmante o responsable de recepción).
- `deliveryReceipt` — recibí firmado. Contiene `signatureImage` (imagen
  de la firma manuscrita en formato data URL) y `signedAt` (fecha y hora
  de la firma).
- `rejectedDate`, `rejectedBy` — fecha y responsable del rechazo, si
  aplica.

Ninguno de estos campos es obligatorio al crear el albarán; se rellenan
al evolucionar el documento.

## Estructura del albarán

Forma común a los documentos de venta de FacturaDirecta:

- `content.type` — siempre `"deliveryNote"`.
- `content.uuid` — identificador inmutable.
- `content.main` — datos del documento (contacto, fechas, divisa,
  líneas, totales, condiciones, plantilla). Puede incluir `owner`, el
  usuario responsable del albarán cuando trabajas con roles personalizados.
- `content.attachments` — adjuntos vinculados (ver [Adjuntos](#adjuntos)).
- `content.meta` — metadatos internos.

En respuestas, además, en el nivel raíz: `tags`, `creationDate`,
`modificationDate`, `related`.

Si creas el albarán con OAuth y no envías `owner`, la API asigna como
responsable al usuario autenticado. Si usas una apiKey, queda sin
responsable salvo que envíes `owner` en el body.

### Líneas de detalle

`content.main.lines` es un array. Cada línea requiere `text`, `quantity`
y `unitPrice`. Otros campos relevantes:

- `discount` o `discountRate` — descuento absoluto o porcentual.
- `tax` — array de **identificadores de impuesto** (`TaxId`) que aplican
  a la línea. Los IDs se obtienen del catálogo de **ventas** de la
  empresa con `GET /{companyId}/settings/taxes/sales`. Ver
  [Impuestos](../guides/taxes.md).
- `lineTotal` — total de la línea; la API lo calcula si lo omites.
- `document` — ID del producto del catálogo si la línea proviene de uno.
- `account` — cuenta contable de la línea; por defecto se toma de la
  cuenta asociada al producto o, en su defecto, la de ventas de la empresa.

**Importes y totales.** Los totales del documento (`total`,
`totalBeforeTaxes`, `linesTotal`, `taxes`) se calculan a partir de las
líneas. Conviene dejarlos vacíos al crear o actualizar y leerlos de la
respuesta.

`taxIncludedPrices: true` indica que `unitPrice` ya incluye IVA. Por
defecto es `false`.

## Operaciones

- [Lista de albaranes](#lista-de-albaranes)
- [Crear albarán](#crear-albarán)
- [Obtener un albarán](#obtener-un-albarán)
- [Actualizar albarán](#actualizar-albarán)
- [Borrar albarán](#borrar-albarán)
- [Actualizar etiquetas de albarán](#actualizar-etiquetas-de-albarán)
- [Enviar albarán por correo](#enviar-albarán-por-correo)
- [Generar el albarán en PDF](#generar-el-albarán-en-pdf)
- [Adjuntos](#adjuntos)

## Lista de albaranes

`GET /{companyId}/deliveryNotes` devuelve los albaranes de la empresa,
paginados.

**Parámetros de consulta principales:**

- `minDate`, `maxDate` — rango de fechas del documento (`YYYY-MM-DD`).
- `series`, `formattedSeries` — serie de numeración.
- `minNumber`, `maxNumber` — rango de número de documento.
- `contact` — ID de contacto.
- `hasContact` — `true`/`false` para filtrar con o sin contacto.
- `minTotal`, `maxTotal` — rango de importe.
- `currency` — código ISO 4217.
- `country` — código país ISO 3166-1 Alpha-2.
- `emails` — buscar por destinatario.
- `allTheseTags`, `anyOfTheseTags`, `hasTags` — filtros por etiquetas.
- `sortBy` — campo de orden.
- `related` — recursos a expandir.

Ver [Paginación](../guides/pagination.md) para los parámetros comunes.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/deliveryNotes?minDate=2026-01-01&limit=50"
```

## Crear albarán

`POST /{companyId}/deliveryNotes` crea un albarán.

**Parámetros del body:**

- `content.type` — siempre `"deliveryNote"`.
- `content.main` — campos del documento. `contact` y `lines` son las
  partes esenciales.
- `tags` (opcional).

**Notas:**

- Si `contact` apunta a un ID existente, sus datos se toman del contacto.
- `series` y `docNumber` se asignan automáticamente si no los envías,
  según la numeración configurada para albaranes.
- La respuesta es el albarán creado completo, en el mismo formato que
  [Obtener un albarán](#obtener-un-albarán).

###### Ejemplo de request JSON

```json
{
  "content": {
    "type": "deliveryNote",
    "main": {
      "docNumber": { "series": "AL" },
      "baseState": "pending",
      "contact": "con_2c4f8e21-7b3a-4d9c-9e1f-a8b7c6d5e4f3",
      "currency": "EUR",
      "lines": [
        {
          "quantity": 1,
          "unitPrice": 50,
          "tax": ["S_IVA_21"],
          "text": "Descripción del producto 1"
        }
      ]
    }
  }
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '@deliveryNote.json' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/deliveryNotes"
```

## Obtener un albarán

`GET /{companyId}/deliveryNotes/{id}` devuelve un albarán por ID.

**Parámetros:**

- `related` (opcional) — recursos relacionados a expandir, por ejemplo
  `invoices` para ver si el albarán está vinculado a alguna factura.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/deliveryNotes/den_3a7b9c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c?related=invoices"
```

## Actualizar albarán

`PUT /{companyId}/deliveryNotes/{id}` sustituye **el contenido completo**
del albarán. No es un PATCH: lo que no envíes, se borra.

Para cambiar el estado del documento, envía el contacto y las líneas
inalterados junto con el nuevo `baseState` (y, si aplica,
`deliveredDate`/`deliveredTo`, `deliveryReceipt` o
`rejectedDate`/`rejectedBy`).

**Restricciones:**

- Si el albarán está vinculado a una factura ya emitida, las
  modificaciones pueden producir efectos secundarios. Antes de
  actualizar, consulta su `related`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '@deliveryNote.json' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/deliveryNotes/den_3a7b9c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c"
```

## Borrar albarán

`DELETE /{companyId}/deliveryNotes/{id}` elimina un albarán.

**Restricciones.** No se puede borrar un albarán vinculado a una factura
emitida. La API devuelve `409 Conflict` con el detalle del documento que
bloquea el borrado.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/deliveryNotes/den_3a7b9c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c"
```

## Actualizar etiquetas de albarán

`PUT /{companyId}/deliveryNotes/{id}/tags` reemplaza el conjunto de
etiquetas sin tocar el resto del contenido.

###### Ejemplo de request JSON

```json
{
  "tags": ["q2-2026", "logística"]
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '{"tags":["q2-2026","logística"]}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/deliveryNotes/den_3a7b9c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c/tags"
```

## Enviar albarán por correo

`PUT /{companyId}/deliveryNotes/{id}/send` envía el albarán por correo
electrónico. Los destinatarios, asunto y cuerpo se toman de la plantilla
configurada para albaranes; los campos del body permiten sobreescribirlos
puntualmente.

**Parámetros del body** (todos opcionales):

- `to`, `cc`, `bcc` — listas de destinatarios.
- `from` — remitente; debe ser una dirección autorizada en la empresa.
- `subject` — asunto.
- `html` — cuerpo del email en HTML.

Si no envías body, se usa enteramente la plantilla del albarán.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '{"to":["recepcion@acme.es"],"subject":"Albarán 2026/AL/0012"}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/deliveryNotes/den_3a7b9c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c/send"
```

## Generar el albarán en PDF

`PUT /{companyId}/deliveryNotes/{id}/pdf` genera una URL temporal con el
albarán en PDF.

**Parámetros del body:**

- `mode` (opcional) — `"attachment"` (URL para descargar como adjunto,
  por defecto), `"inline"` (visualización en línea) o `"print"` (URL
  HTML que carga el PDF y lo envía a imprimir).

La URL devuelta es válida solo durante el periodo indicado en la respuesta.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '{"mode":"inline"}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/deliveryNotes/den_3a7b9c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c/pdf"
```

## Adjuntos

Un albarán puede tener varios adjuntos vinculados (PDFs firmados por el
cliente, documentación de logística, fotos de la entrega...). Los
adjuntos **no se suben en el body del albarán**: primero se crean con
`POST /uploads` y después se vinculan con
`POST /deliveryNotes/{id}/attachments` enviando los `uploadIds`.

### Listar adjuntos

`GET /{companyId}/deliveryNotes/{id}/attachments` devuelve los adjuntos
vinculados al albarán.

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/deliveryNotes/den_3a7b9c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c/attachments"
```

### Vincular adjuntos

`POST /{companyId}/deliveryNotes/{id}/attachments` añade adjuntos al
albarán a partir de uploads previos.

**Parámetros del body:**

- `uploadIds` — array de identificadores `upl_<uuid>` obtenidos con
  `POST /uploads`. Mínimo uno.

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"uploadIds":["upl_6f8e9b42-1d3c-4a7f-b2e4-9c1d5e7f3a6b","upl_7a9b0c53-2e4d-4b8a-a3f5-0d2e6f8a4b7c"]}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/deliveryNotes/den_3a7b9c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c/attachments"
```

### Eliminar un adjunto

`DELETE /{companyId}/deliveryNotes/{id}/attachments/{attachmentIndex}`
elimina un adjunto del albarán. `attachmentIndex` es la posición en el
array (basada en cero).

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/deliveryNotes/den_3a7b9c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c/attachments/0"
```

## Errores comunes

- `400 ValidationError` — falta `contact` o `lines` con `quantity` /
  `unitPrice` inválidos.
- `400 ValidationError` — `baseState` con valor no admitido (debe ser
  `pending`, `delivered`, `closed` o `rejected`).
- `409 Conflict` — borrado o modificación incompatible con una factura
  ya emitida vinculada al albarán.
- `409 Conflict` — `series` o `docNumber` duplicado en la empresa.

Ver [Errores y validaciones](../guides/errors.md) para el formato general
de respuestas de error.

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| GET | `/{companyId}/deliveryNotes` | `getDeliveryNotes` | `deliveryNotes:read` | Lista de albaranes |
| GET | `/{companyId}/deliveryNotes/{id}` | `getDeliveryNote` | `deliveryNotes:read` | Obtener un albarán |
| GET | `/{companyId}/deliveryNotes/{id}/attachments` | `getDeliveryNoteAttachments` | `deliveryNotes:read` | Listar adjuntos de un albarán |
| POST | `/{companyId}/deliveryNotes` | `createDeliveryNote` | `deliveryNotes:write` | Crear albarán |
| POST | `/{companyId}/deliveryNotes/{id}/attachments` | `addDeliveryNoteAttachments` | `deliveryNotes:write` | Vincular adjuntos a un albarán |
| PUT | `/{companyId}/deliveryNotes/{id}` | `updateDeliveryNote` | `deliveryNotes:write` | Actualizar albarán |
| PUT | `/{companyId}/deliveryNotes/{id}/pdf` | `createDeliveryNotePDF` | `deliveryNotes:read` | Generar el albarán en formato PDF |
| PUT | `/{companyId}/deliveryNotes/{id}/send` | `sendDeliveryNote` | `deliveryNotes:write` | Enviar albarán por correo electrónico |
| PUT | `/{companyId}/deliveryNotes/{id}/tags` | `updateDeliveryNoteTags` | `deliveryNotes:write` | Actualizar etiquetas de albarán |
| DELETE | `/{companyId}/deliveryNotes/{id}` | `deleteDeliveryNote` | `deliveryNotes:write` | Borrar albarán |
| DELETE | `/{companyId}/deliveryNotes/{id}/attachments/{attachmentIndex}` | `deleteDeliveryNoteAttachment` | `deliveryNotes:write` | Eliminar adjunto de un albarán |

## Scopes

- **`deliveryNotes:read`** — Lectura de albaranes.
- **`deliveryNotes:write`** — Modificación de albaranes.
