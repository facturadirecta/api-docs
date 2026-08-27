---
title: Presupuestos
audience:
  - developers
status: draft
---

# Presupuestos

Un presupuesto es una propuesta económica que envías a un cliente antes de
emitir la factura correspondiente. Tiene líneas de detalle, totales, condiciones
de pago y un ciclo de vida con estados (`pending`, `accepted`, `rejected`,
`closed`).

Cuando el cliente acepta el presupuesto, lo normal es convertirlo en factura.
**La conversión en sí no es una operación de esta API**: se hace desde la
interfaz. El presupuesto convertido refleja la factura resultante en su
`related` cuando se consulta.

## Estados

`content.main.baseState` controla el estado **declarado** del presupuesto. Es lo
que tú marcas desde el propio documento:

- `pending` — presupuesto enviado o en negociación, sin decisión.
- `accepted` — el cliente lo ha aceptado.
- `rejected` — el cliente lo ha rechazado.
- `closed` — cerrado sin esperar respuesta (por ejemplo, plazo vencido).

El estado **efectivo** que ves en la interfaz puede no coincidir con `baseState`
si existen documentos relacionados. Por ejemplo, un presupuesto `accepted` que
se ha convertido en factura aparece como facturado, aunque su `baseState`
siga siendo `accepted`.

Campos derivados que se rellenan típicamente al cambiar `baseState`:

- `acceptedDate`, `acceptedBy` — quién aceptó y cuándo.
- `rejectedDate`, `rejectedBy` — equivalente para rechazos.

Estos campos no son obligatorios al crear el presupuesto.

## Factura proforma

Un presupuesto puede marcarse como **factura proforma** con
`content.main.isProforma: true`. Una factura proforma es un documento previo a
la factura definitiva que se presenta al cliente con apariencia de factura,
pero sin efectos fiscales reales. Es habitual en escenarios como pagos
anticipados, exportaciones o aprobaciones internas del cliente que exigen un
documento con formato de factura antes de emitir la definitiva.

A nivel del sistema, una factura proforma **sigue siendo un presupuesto**: usa
los mismos endpoints, mismos estados (`baseState`) y la misma estructura de
líneas y totales. Solo cambia su presentación:

- **Nombre del PDF generado** — usa el prefijo "proforma" (o el configurado
  en la plantilla del documento) en vez de "presupuesto".
- **Plantilla de email al enviarlo** — usa la plantilla `email_proforma` (con
  asunto y cuerpo propios) en vez de `email_estimate`. Configura ambas
  plantillas en los ajustes de la empresa si quieres mensajes distintos para
  presupuestos y proformas.
- **Etiquetado interno** — aparece como "proforma" en los listados de la app
  y en los nombres de fichero, no como "presupuesto".

`isProforma` **no implica cambios fiscales**: no genera asientos contables, no
se reporta a VeriFactu/TicketBAI y no consume serie de facturación. Para emitir
la factura real al cliente, crea un nuevo documento de tipo invoice cuando
corresponda.

## Estructura del presupuesto

Forma común a los documentos de venta de FacturaDirecta:

- `content.type` — siempre `"estimate"`.
- `content.uuid` — identificador inmutable.
- `content.main` — datos del documento (contacto, fechas, divisa, líneas,
  totales, condiciones, plantilla). Puede incluir `owner`, el usuario
  responsable del presupuesto cuando trabajas con roles personalizados.
- `content.attachments` — adjuntos vinculados (ver [Adjuntos](#adjuntos)).
- `content.meta` — metadatos internos.

En respuestas, además, en el nivel raíz: `tags`, `creationDate`,
`modificationDate`, `related`.

Si creas el presupuesto con OAuth y no envías `owner`, la API asigna
como responsable al usuario autenticado. Si usas una apiKey, queda sin
responsable salvo que envíes `owner` en el body.

`content.main.customFields` es un mapa de valores indexado por el ID estable
`cfi_<uuid v4>` de cada [campo personalizado](./custom-fields.md). Las claves
deben corresponder a definiciones existentes. También se aceptan definiciones
borradas para conservar presupuestos históricos; un ID desconocido o con
formato incorrecto produce `400 Bad Request`.

### Líneas de detalle

`content.main.lines` es un array. Cada línea requiere `text`, `quantity` y
`unitPrice`. Otros campos relevantes:

- `discount` o `discountRate` — descuento absoluto o porcentual.
- `tax` — array de **identificadores de impuesto** (`TaxId`) que aplican
  a la línea. Los IDs se obtienen del catálogo de **ventas** de la empresa
  con `GET /{companyId}/settings/taxes/sales`. Pueden aplicarse varios
  impuestos a una misma línea, típicamente IVA + retención de IRPF. Ver
  [Impuestos](../guides/taxes.md) para el modelo completo (catálogos,
  autorepercutidos, agregado en el documento).

> En los ejemplos de esta página:
>
> - Los **UUIDs** (`con_2c4f8e21-…`, `est_8d3b6c14-…`, `upl_6f8e9b42-…`) son
>   ilustrativos. Cada empresa tiene los suyos; sustitúyelos por los
>   identificadores reales que devuelve la API en cada caso.
> - Los **IDs de impuestos** (`S_IVA_21`, `S_IRPF_15`) son los del
>   catálogo por defecto del sistema. Pueden variar por empresa o
>   comunidad fiscal; recupera el catálogo real con
>   `GET /{companyId}/settings/taxes/sales`.
- `lineTotal` — total de la línea; la API lo calcula si lo omites.
- `document` — ID del producto del catálogo si la línea proviene de uno.
- `account` — cuenta contable de la línea; por defecto se toma de la cuenta
  asociada al producto o, en su defecto, la de ventas de la empresa.

**Importes y totales.** Los totales del documento (`total`, `totalBeforeTaxes`,
`linesTotal`, `taxes`) se calculan a partir de las líneas. **Conviene dejarlos
vacíos al crear o actualizar** y leerlos de la respuesta. Enviarlos manualmente
solo está justificado para importaciones que fijan totales históricos.

`taxIncludedPrices: true` indica que `unitPrice` ya incluye IVA. Por defecto es
`false`. Cambia el cálculo de `taxes` y `total`.

Por defecto el documento se trata como presupuesto. Para marcarlo como
factura proforma, pon `content.main.isProforma: true` (ver
[Factura proforma](#factura-proforma)).

## Operaciones

- [Lista de presupuestos](#lista-de-presupuestos)
- [Crear presupuesto](#crear-presupuesto)
- [Obtener un presupuesto](#obtener-un-presupuesto)
- [Actualizar presupuesto](#actualizar-presupuesto)
- [Borrar presupuesto](#borrar-presupuesto)
- [Actualizar etiquetas de presupuesto](#actualizar-etiquetas-de-presupuesto)
- [Enviar presupuesto por correo](#enviar-presupuesto-por-correo)
- [Generar el presupuesto en PDF](#generar-el-presupuesto-en-pdf)
- [Adjuntos](#adjuntos)

## Lista de presupuestos

`GET /{companyId}/estimates` devuelve los presupuestos de la empresa, paginados.

**Parámetros de consulta principales:**

- `minDate`, `maxDate` — rango de fechas (`YYYY-MM-DD`).
- `series`, `formattedSeries` — serie de numeración (en formato canónico o como aparece al usuario).
- `minNumber`, `maxNumber` — rango de número de documento.
- `contact` — ID de contacto.
- `hasContact` — `true`/`false` para filtrar con o sin contacto.
- `minTotal`, `maxTotal` — rango de importe total.
- `currency` — código ISO 4217 (`EUR`, `USD`...).
- `country` — código país ISO 3166-1 Alpha-2.
- `emails` — buscar por destinatario.
- `allTheseTags`, `anyOfTheseTags`, `hasTags` — filtros por etiquetas.
- `sortBy` — campo de orden.
- `related` — recursos a expandir.

Ver [Paginación](../guides/pagination.md) para los parámetros comunes
(`offset`, `limit`, `hasMore`).

###### Ejemplo de respuesta JSON

```json
{
  "pagination": { "offset": 0, "limit": 50, "total": 42 },
  "items": [
    {
      "content": {
        "type": "estimate",
        "uuid": "est_8d3b6c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c",
        "main": {
          "title": "Presupuesto 2026/0041 — Acme Diseño",
          "baseState": "pending",
          "docNumber": "2026/0041",
          "series": "default",
          "date": "2026-05-03",
          "dueDate": "2026-06-02",
          "contact": "con_2c4f8e21-7b3a-4d9c-9e1f-a8b7c6d5e4f3",
          "currency": "EUR",
          "lines": [
            {
              "text": "Diseño de identidad corporativa",
              "quantity": 1,
              "unitPrice": 1800.00,
              "tax": ["S_IVA_21"]
            }
          ],
          "linesTotal": 1800.00,
          "totalBeforeTaxes": 1800.00,
          "taxes": [
            {
              "id": "S_IVA_21",
              "base": 1800.00,
              "amount": 378.00,
              "companyCurrencyBase": 1800.00,
              "companyCurrencyAmount": 378.00
            }
          ],
          "total": 2178.00
        }
      },
      "tags": ["q2-2026"],
      "creationDate": "2026-05-03T09:12:00.000Z",
      "modificationDate": "2026-05-03T09:12:00.000Z"
    }
  ]
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/estimates?minDate=2026-01-01&limit=50"
```

## Crear presupuesto

`POST /{companyId}/estimates` crea un presupuesto.

**Parámetros del body:**

- `content.type` — siempre `"estimate"`.
- `content.main` — campos del documento. `lines` es lo importante; los totales
  se calculan a partir de ellas.
- `tags` (opcional).

**Notas:**

- Si `contact` apunta a un ID existente, sus datos fiscales y de envío se
  toman de ese contacto.
- `series` y `docNumber` se asignan automáticamente si no los envías, según la
  numeración configurada para presupuestos.
- La respuesta es el presupuesto creado completo, en el formato de
  [Obtener un presupuesto](#obtener-un-presupuesto).

###### Ejemplo de request JSON

```json
{
  "content": {
    "type": "estimate",
    "main": {
      "contact": "con_2c4f8e21-7b3a-4d9c-9e1f-a8b7c6d5e4f3",
      "date": "2026-05-03",
      "dueDate": "2026-06-02",
      "currency": "EUR",
      "lines": [
        {
          "text": "Diseño de identidad corporativa",
          "quantity": 1,
          "unitPrice": 1800.00,
          "tax": ["S_IVA_21"]
        }
      ]
    }
  },
  "tags": ["q2-2026"]
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{
    "content": {
      "type": "estimate",
      "main": {
        "contact": "con_2c4f8e21-7b3a-4d9c-9e1f-a8b7c6d5e4f3",
        "date": "2026-05-03",
        "currency": "EUR",
        "lines": [
          { "text": "Diseño de identidad corporativa", "quantity": 1, "unitPrice": 1800.00, "tax": ["S_IVA_21"] }
        ]
      }
    }
  }' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/estimates"
```

## Obtener un presupuesto

`GET /{companyId}/estimates/{id}` devuelve un presupuesto por ID.

**Parámetros:**

- `related` (opcional) — recursos relacionados a expandir, por ejemplo
  `invoices` para ver si el presupuesto ha derivado en factura.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/estimates/est_8d3b6c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c?related=invoices"
```

## Actualizar presupuesto

`PUT /{companyId}/estimates/{id}` sustituye **el contenido completo** del
presupuesto. No es un PATCH: lo que no envíes, se borra.

Para cambiar el estado del documento, envía el contacto y las líneas
inalterados junto con el nuevo `baseState` (y, si aplica, `acceptedDate` /
`rejectedDate`).

**Restricciones:**

- Si el presupuesto ya está convertido en una factura, las modificaciones
  pueden estar limitadas o producir efectos secundarios. Antes de actualizar
  un `accepted`, consulta su `related`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '@estimate.json' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/estimates/est_8d3b6c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c"
```

## Borrar presupuesto

`DELETE /{companyId}/estimates/{id}` elimina un presupuesto.

**Restricciones.** No se puede borrar un presupuesto que ya esté vinculado a
una factura emitida. La API devuelve `409 Conflict` con el detalle del documento
que bloquea el borrado.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/estimates/est_8d3b6c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c"
```

## Actualizar etiquetas de presupuesto

`PUT /{companyId}/estimates/{id}/tags` reemplaza el conjunto de etiquetas sin
tocar el resto del contenido.

###### Ejemplo de request JSON

```json
{
  "tags": ["q2-2026", "agencia", "prioridad-alta"]
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '{"tags":["q2-2026","agencia","prioridad-alta"]}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/estimates/est_8d3b6c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c/tags"
```

## Enviar presupuesto por correo

`PUT /{companyId}/estimates/{id}/send` envía el presupuesto por correo
electrónico. Los destinatarios, asunto y cuerpo se toman de la plantilla
configurada para presupuestos; los campos del body permiten **sobreescribirlos**
puntualmente.

**Parámetros del body** (todos opcionales):

- `to`, `cc`, `bcc` — listas de destinatarios; sobrescriben los valores de la plantilla.
- `from` — remitente; debe ser una dirección autorizada en la empresa.
- `subject` — asunto.
- `html` — cuerpo del email en HTML.

Si no envías body, se usa enteramente la plantilla del presupuesto.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '{"to":["compras@acme.es"],"subject":"Presupuesto 2026/0041"}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/estimates/est_8d3b6c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c/send"
```

## Generar el presupuesto en PDF

`PUT /{companyId}/estimates/{id}/pdf` genera una URL temporal con el
presupuesto en PDF.

**Parámetros del body:**

- `mode` (opcional) — `"attachment"` (URL para descargar como adjunto, por
  defecto), `"inline"` (visualización en línea) o `"print"` (URL HTML que
  carga el PDF y lo envía a imprimir).

La URL devuelta es válida solo durante el periodo indicado en la respuesta.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '{"mode":"inline"}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/estimates/est_8d3b6c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c/pdf"
```

## Adjuntos

Un presupuesto puede tener varios adjuntos vinculados (PDFs de referencia,
documentación técnica, etc.). Los adjuntos **no se suben en el body del
presupuesto**: primero se crean con `POST /uploads` y después se vinculan al
presupuesto con `POST /estimates/{id}/attachments` enviando los `uploadIds`.

Una vez vinculados, los adjuntos viven dentro del recurso y se pueden listar o
eliminar de forma independiente.

### Listar adjuntos

`GET /{companyId}/estimates/{id}/attachments` devuelve los adjuntos vinculados
al presupuesto.

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/estimates/est_8d3b6c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c/attachments"
```

### Vincular adjuntos

`POST /{companyId}/estimates/{id}/attachments` añade adjuntos al presupuesto a
partir de uploads previos.

**Parámetros del body:**

- `uploadIds` — array de identificadores `upl_<uuid>` obtenidos con
  `POST /uploads`. Mínimo uno.

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"uploadIds":["upl_6f8e9b42-1d3c-4a7f-b2e4-9c1d5e7f3a6b","upl_7a9b0c53-2e4d-4b8a-a3f5-0d2e6f8a4b7c"]}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/estimates/est_8d3b6c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c/attachments"
```

### Eliminar un adjunto

`DELETE /{companyId}/estimates/{id}/attachments/{attachmentIndex}` elimina un
adjunto del presupuesto. `attachmentIndex` es la posición del adjunto en el
array (basada en cero).

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/estimates/est_8d3b6c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c/attachments/0"
```

## Errores comunes

- `400 ValidationError` — falta `contact`, o líneas con `quantity` o `unitPrice` inválidos.
- `400 ValidationError` — `baseState` con valor no admitido.
- `409 Conflict` — borrado o modificación incompatible con un documento relacionado (típicamente, factura ya emitida).
- `409 Conflict` — `series` o `docNumber` duplicado en la empresa.

Ver [Errores y validaciones](../guides/errors.md) para el formato general de
respuestas de error.

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| GET | `/{companyId}/estimates` | `getEstimates` | `estimates:read` | Lista de presupuestos |
| GET | `/{companyId}/estimates/{id}` | `getEstimate` | `estimates:read` | Obtener un presupuesto |
| GET | `/{companyId}/estimates/{id}/attachments` | `getEstimateAttachments` | `estimates:read` | Listar adjuntos de un presupuesto |
| POST | `/{companyId}/estimates` | `createEstimate` | `estimates:write` | Crear presupuesto |
| POST | `/{companyId}/estimates/{id}/attachments` | `addEstimateAttachments` | `estimates:write` | Vincular adjuntos a un presupuesto |
| PUT | `/{companyId}/estimates/{id}` | `updateEstimate` | `estimates:write` | Actualizar presupuesto |
| PUT | `/{companyId}/estimates/{id}/pdf` | `createEstimatePDF` | `estimates:read` | Generar el presupuesto en formato PDF |
| PUT | `/{companyId}/estimates/{id}/send` | `sendEstimate` | `estimates:write` | Enviar presupuesto por correo electrónico |
| PUT | `/{companyId}/estimates/{id}/tags` | `updateEstimateTags` | `estimates:write` | Actualizar etiquetas de presupuesto |
| DELETE | `/{companyId}/estimates/{id}` | `deleteEstimate` | `estimates:write` | Borrar presupuesto |
| DELETE | `/{companyId}/estimates/{id}/attachments/{attachmentIndex}` | `deleteEstimateAttachment` | `estimates:write` | Eliminar adjunto de un presupuesto |

## Scopes

- **`estimates:read`** — Lectura de presupuestos.
- **`estimates:write`** — Modificación de presupuestos.
