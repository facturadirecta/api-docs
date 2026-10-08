---
title: Órdenes de compra
audience:
  - developers
status: draft
---

# Órdenes de compra

Una orden de compra registra un encargo en firme a un proveedor antes de la
recepción de la mercancía y su facturación. Tiene líneas de detalle, totales y
condiciones como el resto de documentos de compra, con una diferencia clave:
**cada línea tiene su propio estado** (pendiente de recibir, recibida o
cancelada), y el estado de la orden se calcula a partir de ellas.

Cuando la mercancía se recibe, lo habitual es convertir la orden en factura de
compra. **La conversión no es una operación de esta API**: se hace desde la
interfaz. El gasto resultante conserva el enlace con la orden a través del
campo `origin` de sus líneas.

La orden de compra es además el lado que **manda** en la coordinación con el
[pedido de cliente](./client-orders.md): los vínculos línea a línea se crean
desde aquí y los estados se propagan desde aquí. Ver
[Flujo de pedidos y órdenes de compra](../guides/orders-flow.md).

> En los ejemplos de esta página los **UUIDs** (`pur_…`, `con_…`, `upl_…`) son
> ilustrativos: sustitúyelos por los identificadores reales de tu empresa. Los
> **IDs de impuestos** (`P_IVA_21_SV`) son los del catálogo por defecto;
> recupera los tuyos con `GET /{companyId}/settings/taxes/purchases` y consulta
> [Impuestos](../guides/taxes.md).

## Disponibilidad por plan

Las órdenes de compra y los pedidos forman parte del **Módulo Inventario**.
Están incluidos en el plan **Diamante** y se pueden contratar como módulo en
**Bronce**, **Plata** y **Oro**. En el plan **Gratis** no están disponibles.

Sin esa disponibilidad, **las operaciones de escritura** (crear, actualizar,
etiquetar y borrar) responden `403 Forbidden` con el código
`plan_limit_exceeded` y el mensaje «Los pedidos de cliente y las órdenes de
compra no están disponibles en tu plan». Las
operaciones de lectura no fallan: devuelven lo que haya.

## Estados

### Estado de la orden (calculado)

`content.main.state` es el estado de la orden y es de **solo lectura en la
práctica**: FacturaDirecta lo recalcula en cada guardado a partir del estado de
las líneas, con esta prioridad:

- `pending` — alguna línea está pendiente de recibir.
- `received` — sin líneas pendientes y alguna línea recibida.
- `canceled` — todas las líneas están canceladas.

Si envías `main.state` en un create o un update, el valor se ignora y se
sustituye por el calculado. Para cambiar el estado de la orden, cambia el
estado de sus líneas.

Como en los pedidos, las **líneas vacías** (sin texto ni importe) no cuentan, y
una orden sin líneas con contenido queda en `pending`.

### Estado de línea

`content.main.lines[].state` es escribible y admite estos valores:

- `pending` — pendiente de recibir.
- `received` — recibida.
- `canceled` — cancelada.

Las líneas nuevas sin `state` se crean como `pending`. Cambiar el estado de una
línea vinculada a un pedido de cliente **modifica también el pedido**: ver
[Flujo de pedidos y órdenes de compra](../guides/orders-flow.md).

## Identidad de las líneas

Cada línea tiene un `id` **estable** que la identifica dentro de la orden.
FacturaDirecta lo genera si no lo envías.

Al **actualizar** una orden las líneas se sustituyen por las del body, con esta
regla: una línea que incluya un `id` ya existente **conserva su identidad** (y
con ella los vínculos que esa línea tenga con otros documentos); una línea sin
`id` se trata como línea nueva. Si actualizas órdenes por API, conserva los
`id` que devolvió la API en las líneas que no quieras recrear.

Los `id` deben ser únicos dentro del documento: dos líneas con el mismo `id`
se rechazan con `400`.

## Estructura de la orden de compra

- `content.type` — siempre `"purchaseOrder"`.
- `content.uuid` — identificador inmutable, prefijo `pur_`.
- `content.main` — datos del documento (proveedor, fechas, divisa, líneas,
  totales, plantilla).
- `content.attachments` — adjuntos vinculados (ver [Adjuntos](#adjuntos)).
- `content.meta` — metadatos internos.

Dentro de `content.main` son obligatorios `docNumber` y `lines`. Campos con
comportamiento propio:

| Campo | Significado |
|---|---|
| `contact` | El **proveedor**. Los impuestos y las cuentas contables se resuelven con el catálogo de compras. |
| `state` | Estado de la orden. Calculado; se ignora al escribir. |
| `dueDate` | Fecha de recepción prevista de la mercancía. |
| `warehouse` | Almacén del documento a efectos de stock. Si no lo indicas se usa el almacén por defecto de la empresa. |
| `customFields` | Valores de campos personalizados del documento. |
| `counterpart` | Datos fiscales de la contraparte. |

A diferencia del pedido de cliente, la orden de compra **no tiene `owner`**.

En respuestas, además, en el nivel raíz: `tags`, `creationDate`,
`modificationDate` y `related`. **`related` solo transporta los objetos que
pidas con el parámetro `related`** (hoy, definiciones de impuestos): no
contiene los documentos en los que se haya convertido la orden.

### Líneas de detalle

`content.main.lines` es un array. Cada línea requiere `text`, `quantity` y
`unitPrice`. Además de los campos comunes a los documentos de compra
(`discount`/`discountRate`, `tax`, `document`, `account`, `lineTotal`), las
líneas de la orden añaden:

- `id` — identidad estable de la línea (ver
  [Identidad de las líneas](#identidad-de-las-líneas)).
- `state` — estado de la línea (ver [Estado de línea](#estado-de-línea)).
- `client` — ID del contacto cliente para el que se compra esa línea
  (trazabilidad de compras bajo pedido de cliente).
- `clientOrder`, `clientOrderLine` — vínculo con el pedido de cliente y con la
  línea concreta que origina esta línea. **Este es el lado desde el que se
  crea el vínculo**: al guardar la orden, FacturaDirecta completa el vínculo
  recíproco en el pedido y propaga el estado. Para que el vínculo se acepte, el
  proveedor de la línea de pedido debe coincidir con el proveedor de la orden,
  y el producto y la cantidad deben ser los mismos en las dos líneas.

Si omites `tax` en una línea, FacturaDirecta resuelve el impuesto por defecto a
partir del producto, del proveedor y de la posición fiscal del documento.

Las líneas **con producto** deben llevar `quantity` mayor que cero, y ese
producto debe tener activada la faceta de compra (`purchases`).

**Importes y totales.** Los totales del documento (`total`,
`totalBeforeTaxes`, `linesTotal`, `taxes`) se calculan a partir de las líneas.
Conviene dejarlos vacíos al crear o actualizar y leerlos de la respuesta.

## Enlace a la aplicación

Las respuestas que devuelven una orden de compra incluyen `webUrl` junto al resto de
los datos. En los listados, cada elemento de `items` lleva su propio enlace.

`webUrl` abre la ficha del elemento en la aplicación web de FacturaDirecta.
Puedes mostrarlo como enlace en tu integración sin construir rutas internas.
El navegador pedirá iniciar sesión con un usuario que tenga acceso a la
empresa. Es un campo opcional y no es un endpoint de la API: trátalo como una
URL para el usuario, no como una URL a la que enviar `$ACCESS_TOKEN`.

## Operaciones

- [Lista de órdenes de compra](#lista-de-órdenes-de-compra)
- [Crear orden de compra](#crear-orden-de-compra)
- [Obtener una orden de compra](#obtener-una-orden-de-compra)
- [Actualizar orden de compra](#actualizar-orden-de-compra)
- [Borrar orden de compra](#borrar-orden-de-compra)
- [Actualizar etiquetas](#actualizar-etiquetas)
- [Enviar la orden por correo](#enviar-la-orden-por-correo)
- [Generar la orden en PDF](#generar-la-orden-en-pdf)
- [Adjuntos](#adjuntos)

## Lista de órdenes de compra

`GET /{companyId}/purchaseOrders` devuelve las órdenes activas de la empresa,
paginadas.

**Parámetros de consulta específicos:**

- **`state`** — estado de la orden: `pending`, `received` o `canceled`. Repite
  el parámetro para incluir varios estados.
- **`minDate`** / **`maxDate`** — fecha de la orden (`YYYY-MM-DD`), inclusive.
- **`series`** — serie sin aplicar el formato de año (`##` o `####`).
- **`formattedSeries`** — serie ya formateada.
- **`minNumber`** / **`maxNumber`** — número secuencial, inclusive.
- **`contact`** — proveedor de la orden. Admite varios valores.
- **`hasContact`** — `false` para obtener las órdenes sin contacto asociado.
- **`minTotal`** / **`maxTotal`** — importe total.
- **`currency`** — moneda (ISO 4217).
- **`country`** — país de la dirección fiscal del proveedor (ISO 3166-1
  alpha-2).
- **`emails`** — búsqueda parcial en los correos del documento, insensible a
  mayúsculas y acentos; todas las palabras deben coincidir.
- **`allTheseTags`** — la orden debe llevar todas las etiquetas indicadas.
- **`anyOfTheseTags`** — basta con una de las etiquetas indicadas.
- **`hasTags`** — `true` devuelve solo órdenes con al menos una etiqueta y `false`, solo órdenes sin etiquetas.
- **`sortBy`** — orden de los resultados. Valores: `date`, `series`,
  `formattedSeries`, `number`, `total`, `currency`, `country`, `creationDate`,
  `modificationDate`. Prefija con `-` para orden descendente y repite el
  parámetro para ordenar por varios criterios.
- **`related`** — datos adicionales. Único valor disponible: `taxIds`, que
  devuelve en `related.objects` las definiciones de los impuestos usados.

**Parámetros globales aceptados:**

Acepta además los parámetros estándar `offset`, `limit`, `minCreationDate`,
`maxCreationDate`, `minModificationDate`, `maxModificationDate` y el header
`accept-version`. Ver [Paginación](../guides/pagination.md) y
[Autenticación](../guides/authentication.md).

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/purchaseOrders?state=pending&sortBy=-date&limit=50"
```

## Crear orden de compra

`POST /{companyId}/purchaseOrders` crea una orden de compra.

**Parámetros del body:**

- `content` (obligatorio) — la orden. `content.type` debe ser
  `"purchaseOrder"` y `content.main` debe llevar `docNumber` y `lines`.
- `tags` (opcional) — etiquetas iniciales de la orden.

**Notas:**

- Si no envías `content.uuid`, la API asigna uno con prefijo `pur_`. Si lo
  envías y ya existe, responde `409 Conflict`.
- El `state` de la orden se calcula desde las líneas: lo que envíes en
  `main.state` se descarta.
- Para abastecer líneas de pedido, indica `clientOrder` y `clientOrderLine` en
  cada línea. Al guardar, esas líneas de pedido pasan a `ordered`.

**Parámetros globales aceptados:** `accept-version`.

###### Ejemplo de request JSON

Contenido mínimo (ejemplo `minimal` del openapi). Las líneas se crean
pendientes de recibir, así que la orden queda `pending`:

```json
{
  "content": {
    "type": "purchaseOrder",
    "main": {
      "docNumber": { "series": "OC" },
      "contact": "con_7b1d2c93-4e5f-4a6b-8c7d-9e0f1a2b3c4d",
      "currency": "EUR",
      "lines": [
        {
          "quantity": 1,
          "unitPrice": 100,
          "tax": ["P_IVA_21_SV"],
          "text": "Descripción del artículo a comprar"
        }
      ]
    }
  }
}
```

Con estado por línea (ejemplo `conEstadosDeLinea`). Todas las líneas están
recibidas, así que la orden queda `received`:

```json
{
  "content": {
    "type": "purchaseOrder",
    "main": {
      "docNumber": { "series": "OC" },
      "contact": "con_7b1d2c93-4e5f-4a6b-8c7d-9e0f1a2b3c4d",
      "currency": "EUR",
      "lines": [
        {
          "quantity": 2,
          "unitPrice": 50,
          "tax": ["P_IVA_21_SV"],
          "text": "Artículo ya recibido del proveedor",
          "state": "received"
        },
        {
          "quantity": 1,
          "unitPrice": 75,
          "tax": ["P_IVA_21_SV"],
          "text": "Otro artículo ya recibido",
          "state": "received"
        }
      ]
    }
  }
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '@purchaseOrder.json' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/purchaseOrders"
```

## Obtener una orden de compra

`GET /{companyId}/purchaseOrders/{id}` devuelve una orden por ID.

**Parámetros de consulta específicos:**

- **`related`** — `taxIds` para recibir en `related.objects` las definiciones
  de los impuestos usados en la orden.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/purchaseOrders/pur_9f8e7d66-5c4b-4a39-8271-6e5d4c3b2a10?related=taxIds"
```

## Actualizar orden de compra

`PUT /{companyId}/purchaseOrders/{id}` sustituye **el contenido completo** de
la orden. No es un PATCH: lo que no envíes se pierde.

**Parámetros del body:**

- `content` (obligatorio) — la orden completa. Si incluye `uuid`, debe
  coincidir con el de la ruta.
- `syncContactData` (opcional) — con `true`, actualiza dirección
  (`address`, `country`, `zipcode`) y `counterpart` a partir del contacto de
  `main.contact`. No toca correos, posición fiscal, vencimiento ni cuenta
  contable.
- `tags` y `tagsOperation` (opcionales) — etiquetas y operación a aplicar
  (`add`, `remove` o `replace`).

**Notas:**

- Conserva los `id` de las líneas que no quieras recrear.
- Cada guardado vuelve a propagar los estados a los pedidos vinculados, que se
  guardan también y emiten su propio `client_order.updated`.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '@purchaseOrder.json' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/purchaseOrders/pur_9f8e7d66-5c4b-4a39-8271-6e5d4c3b2a10"
```

## Borrar orden de compra

`DELETE /{companyId}/purchaseOrders/{id}` borra la orden. El borrado es
recuperable desde la interfaz; el documento queda con `archived: true` y emite
el evento `purchase_order.archived`.

Al borrarla, las líneas de pedido que abastecía vuelven a `pending` y quedan
desvinculadas. Los vínculos se conservan del lado de la orden para poder
rehacerlos si se recupera.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/purchaseOrders/pur_9f8e7d66-5c4b-4a39-8271-6e5d4c3b2a10"
```

## Actualizar etiquetas

`PUT /{companyId}/purchaseOrders/{id}/tags` cambia solo las etiquetas, sin
tocar el contenido de la orden.

**Parámetros del body** (ambos obligatorios):

- `tags` — array de etiquetas.
- `tagsOperation` — `add`, `remove` o `replace`.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '{"tags":["reposicion"],"tagsOperation":"add"}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/purchaseOrders/pur_9f8e7d66-5c4b-4a39-8271-6e5d4c3b2a10/tags"
```

## Enviar la orden por correo

`PUT /{companyId}/purchaseOrders/{id}/send` envía la orden al proveedor por
correo electrónico. Los destinatarios, el asunto y el cuerpo se toman de la
plantilla del tema; los campos del body permiten sobreescribirlos.

**Parámetros del body** (todos opcionales):

- `to`, `cc`, `bcc` — listas de destinatarios; sustituyen a los valores de
  `main.emails`, `main.emailsCc` y `main.emailsBcc` del documento.
- `from` — remitente; debe ser una dirección dada de alta en la empresa.
- `subject` — asunto.
- `html` — cuerpo del mensaje en HTML.

Debe haber al menos un destinatario. Si no se puede determinar ninguno, o si la
empresa no tiene remitentes configurados, la respuesta es `409 Conflict`.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '{"to":["pedidos@proveedor.es"],"subject":"Orden de compra OC-2026-08"}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/purchaseOrders/pur_9f8e7d66-5c4b-4a39-8271-6e5d4c3b2a10/send"
```

## Generar la orden en PDF

`PUT /{companyId}/purchaseOrders/{id}/pdf` genera una URL temporal con la orden
en PDF.

**Parámetros del body:**

- `mode` — `"attachment"` (descarga como adjunto), `"inline"` (visualización en
  línea) o `"print"` (URL HTML que carga el PDF y lo envía a imprimir). Un
  valor ausente o distinto de esos tres devuelve `400`.

La respuesta lleva `url`, `filename` y `availableUntil`; en modo `print` añade
`urlPdf`. La URL es válida hasta la fecha de `availableUntil` (tres días).

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '{"mode":"inline"}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/purchaseOrders/pur_9f8e7d66-5c4b-4a39-8271-6e5d4c3b2a10/pdf"
```

## Adjuntos

Una orden de compra puede tener hasta **10 adjuntos**. Los adjuntos no se
suben en el body de la orden: primero se crean con `POST /uploads` y después se
vinculan enviando sus `uploadIds`.

### Listar adjuntos

`GET /{companyId}/purchaseOrders/{id}/attachments` devuelve los adjuntos
vinculados a la orden.

**Parámetros globales aceptados:** `accept-version`.

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/purchaseOrders/pur_9f8e7d66-5c4b-4a39-8271-6e5d4c3b2a10/attachments"
```

### Vincular adjuntos

`POST /{companyId}/purchaseOrders/{id}/attachments` añade adjuntos a la orden a
partir de uploads previos.

**Parámetros del body:**

- `uploadIds` (obligatorio) — array no vacío de identificadores `upl_<uuid>`
  obtenidos con `POST /uploads`.

Si la suma de adjuntos existentes y nuevos supera 10, la respuesta es `400`
con el código `attachments_limit_exceeded`.

**Parámetros globales aceptados:** `accept-version`.

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"uploadIds":["upl_8a2c1234-1234-4123-8123-123456789012"]}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/purchaseOrders/pur_9f8e7d66-5c4b-4a39-8271-6e5d4c3b2a10/attachments"
```

### Eliminar un adjunto

`DELETE /{companyId}/purchaseOrders/{id}/attachments/{attachmentIndex}` elimina
un adjunto. `attachmentIndex` es su posición en el array, empezando en cero;
un valor no entero o negativo devuelve `400`.

**Parámetros globales aceptados:** `accept-version`.

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/purchaseOrders/pur_9f8e7d66-5c4b-4a39-8271-6e5d4c3b2a10/attachments/0"
```

## Numeración

Las órdenes de compra usan sus propias series de numeración, configurables en
los ajustes de la empresa. Si no indicas `docNumber.number` al crear, se asigna
el siguiente número de la serie. Las series con `##` o `####` se expanden con
el año en `docNumber.formattedSeries`.

## Concurrencia

Los guardados y borrados de órdenes de compra de una misma empresa **se
serializan**: cada uno puede tocar los pedidos que abastece, y dos a la vez
decidirían la propagación sobre una foto obsoleta. Si ya hay 20 escrituras de
órdenes en curso en la misma empresa, la siguiente responde
`429 Too Many Requests`. Si cargas órdenes en lote, hazlo en serie o con poca
concurrencia y reintenta ante un `429`.

## Webhooks

Los cambios en órdenes de compra emiten `purchase_order.created`,
`purchase_order.updated`, `purchase_order.archived` y
`purchase_order.unarchived`. Ver [Webhooks](./webhooks.md).

Cada guardado o borrado que afecte a pedidos vinculados emite además un
`client_order.updated` por cada pedido afectado.

## Versiones

`GET /{companyId}/purchaseOrders/{id}/versions` lista las versiones de la orden de compra: su alta, cada
modificación, su eliminación y su recuperación, con quién y cuándo, aunque se haya eliminado.
`GET /{companyId}/purchaseOrders/{id}/versions/{versionId}` devuelve la copia de una versión, en el
mismo formato que `GET /{companyId}/purchaseOrders/{id}`, y los cambios respecto a la versión anterior.
Sirven para ver qué se cambió y recuperar datos: ver
[Versiones de documentos](../guides/document-versions.md).

## Errores comunes

- `400` — `minDate` o `maxDate` es una fecha que no existe, como
  `2026-02-30`. Ver [Fechas que no existen](../guides/errors.md#fechas-que-no-existen-400).
- `400` — el documento tiene referencias que no existen (proveedor, producto,
  método de pago o plantilla).
- `400` — dos líneas con el mismo `id`, vínculo con pedido incompleto
  (documento vinculado y línea vinculada deben ir juntos) o dos líneas
  apuntando a la misma línea de pedido (el vínculo es 1:1).
- `400` — la línea de pedido a la que apuntas no existe, ya está vinculada a
  otra orden de compra, tiene otro proveedor, otro producto u otra cantidad. Si
  necesitas comprar unidades adicionales, añade una línea de reposición
  independiente en vez de cambiar la cantidad de la línea vinculada.
- `400` — línea con producto y `quantity` menor o igual que cero.
- `400` — producto sin las preferencias de compra activadas.
- `400` con código `attachments_limit_exceeded` — se supera el máximo de
  adjuntos del documento.
- `403` — el plan no incluye pedidos ni órdenes de compra (ver
  [Disponibilidad por plan](#disponibilidad-por-plan)), o faltan scopes.
- `404` — la orden no existe en la empresa indicada.
- `409` — `content.uuid` ya existe al crear, o no hay destinatario ni
  remitente al enviar por correo.
- `429` — ya hay 20 escrituras de órdenes en curso en la misma empresa (ver
  [Concurrencia](#concurrencia)).

Ver [Errores y validaciones](../guides/errors.md) para el formato general.

## Referencia exhaustiva

Esta página cubre los matices funcionales y los casos típicos. Para la
referencia exhaustiva de todos los campos del body y la respuesta, consulta el
[Swagger UI](https://www.facturadirecta.com/api) o el
[openapi crudo](https://app.facturadirecta.com/openapi.json).

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| GET | `/{companyId}/purchaseOrders` | `getPurchaseOrders` | `purchaseOrders:read` | Lista de órdenes de compra |
| GET | `/{companyId}/purchaseOrders/{id}` | `getPurchaseOrder` | `purchaseOrders:read` | Obtener una orden de compra |
| GET | `/{companyId}/purchaseOrders/{id}/attachments` | `getPurchaseOrderAttachments` | `purchaseOrders:read` | Listar adjuntos de una orden de compra |
| GET | `/{companyId}/purchaseOrders/{id}/versions` | `getPurchaseOrderVersions` | `purchaseOrders:read` | Versiones de la orden de compra |
| GET | `/{companyId}/purchaseOrders/{id}/versions/{versionId}` | `getPurchaseOrderVersion` | `purchaseOrders:read` | Versión de la orden de compra |
| POST | `/{companyId}/purchaseOrders` | `createPurchaseOrder` | `purchaseOrders:write` | Crear orden de compra |
| POST | `/{companyId}/purchaseOrders/{id}/attachments` | `addPurchaseOrderAttachments` | `purchaseOrders:write` | Vincular adjuntos a una orden de compra |
| PUT | `/{companyId}/purchaseOrders/{id}` | `updatePurchaseOrder` | `purchaseOrders:write` | Actualizar orden de compra |
| PUT | `/{companyId}/purchaseOrders/{id}/pdf` | `createPurchaseOrderPDF` | `purchaseOrders:read` | Generar la orden de compra en formato PDF |
| PUT | `/{companyId}/purchaseOrders/{id}/send` | `sendPurchaseOrder` | `purchaseOrders:write` | Enviar orden de compra por correo electrónico |
| PUT | `/{companyId}/purchaseOrders/{id}/tags` | `updatePurchaseOrderTags` | `purchaseOrders:write` | Actualizar etiquetas de orden de compra |
| DELETE | `/{companyId}/purchaseOrders/{id}` | `deletePurchaseOrder` | `purchaseOrders:write` | Borrar orden de compra |
| DELETE | `/{companyId}/purchaseOrders/{id}/attachments/{attachmentIndex}` | `deletePurchaseOrderAttachment` | `purchaseOrders:write` | Eliminar adjunto de una orden de compra |

## Scopes

- **`purchaseOrders:read`** — Lectura de órdenes de compra.
- **`purchaseOrders:write`** — Modificación de órdenes de compra.
