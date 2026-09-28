---
title: Pedidos
audience:
  - developers
status: draft
---

# Pedidos

Un pedido de cliente registra un encargo en firme de un cliente antes de su
entrega y facturación. Tiene líneas de detalle, totales y condiciones como el
resto de documentos de venta, con una diferencia clave: **cada línea tiene su
propio estado** (pendiente, pedida a proveedor, lista para entregar, entregada
o cancelada), y el estado del pedido se calcula a partir de ellas.

Cuando el pedido está listo, lo habitual es convertirlo en albarán o factura.
**La conversión no es una operación de esta API**: se hace desde la interfaz.
El documento resultante conserva el enlace con el pedido a través del campo
`origin` de sus líneas.

El pedido tiene su contrapartida en compras: la
[orden de compra](./purchase-orders.md). Las dos se coordinan **línea a línea**
y el detalle de esa coordinación está en
[Flujo de pedidos y órdenes de compra](../guides/orders-flow.md).

> En los ejemplos de esta página los **UUIDs** (`cor_…`, `con_…`, `upl_…`) son
> ilustrativos: sustitúyelos por los identificadores reales de tu empresa. Los
> **IDs de impuestos** (`S_IVA_21`) son los del catálogo por defecto; recupera
> los tuyos con `GET /{companyId}/settings/taxes/sales` y consulta
> [Impuestos](../guides/taxes.md).

## Disponibilidad por plan

Los pedidos y las órdenes de compra forman parte del **Módulo Inventario**.
Están incluidos en el plan **Diamante** y se pueden contratar como módulo en
**Bronce**, **Plata** y **Oro**. En el plan **Gratis** no están disponibles.

Sin esa disponibilidad, **las operaciones de escritura** (crear, actualizar,
etiquetar y borrar) responden `403 Forbidden` con el código
`plan_limit_exceeded` y el mensaje «Los pedidos de cliente y las órdenes de
compra no están disponibles en tu plan». Las
operaciones de lectura no fallan: devuelven lo que haya (normalmente, una lista
vacía).

## Estados

### Estado del pedido (calculado)

`content.main.state` es el estado del pedido y es de **solo lectura en la
práctica**: FacturaDirecta lo recalcula en cada guardado a partir del estado de
las líneas, con esta prioridad:

- `pending` — alguna línea está `pending` u `ordered`.
- `ready` — sin líneas pendientes y alguna línea `ready`.
- `delivered` — sin pendientes ni listas y alguna línea `delivered`.
- `canceled` — todas las líneas están canceladas.

Si envías `main.state` en un create o un update, el valor se ignora y se
sustituye por el calculado. Para cambiar el estado del pedido, cambia el estado
de sus líneas.

Dos matices del cálculo:

- Las **líneas vacías** (sin texto ni importe) no cuentan. Puedes usarlas como
  separadores sin alterar el estado.
- Un pedido **sin líneas con contenido** queda en `pending`, que es el estado
  neutro mientras se edita.

### Estado de línea

`content.main.lines[].state` es escribible y admite estos valores:

- `pending` — pendiente: aún no se puede servir.
- `ordered` — pedida a proveedor. Este estado lo fija FacturaDirecta cuando la
  línea queda vinculada a una orden de compra; no lo asignes a mano.
- `ready` — lista para entregar.
- `delivered` — entregada al cliente.
- `canceled` — cancelada.

Las líneas nuevas sin `state` se crean como `pending`.

En una **línea vinculada a una orden de compra**, los estados `pending`,
`ordered` y `ready` los gobierna la orden: si envías uno distinto del que hay
guardado, el servidor restaura el suyo sin error. `delivered` y `canceled` sí
los decides tú desde el pedido y la orden nunca los pisa.

## Identidad de las líneas

Cada línea tiene un `id` **estable** que la identifica dentro del pedido.
FacturaDirecta lo genera si no lo envías.

Al **actualizar** un pedido las líneas se sustituyen por las del body, con esta
regla: una línea que incluya un `id` ya existente **conserva su identidad** (y
con ella los vínculos que esa línea tenga con otros documentos); una línea sin
`id` se trata como línea nueva. Si actualizas pedidos por API, conserva los
`id` que devolvió la API en las líneas que no quieras recrear.

Los `id` deben ser únicos dentro del documento: dos líneas con el mismo `id`
se rechazan con `400`.

## Estructura del pedido

Forma común a los documentos de venta de FacturaDirecta:

- `content.type` — siempre `"clientOrder"`.
- `content.uuid` — identificador inmutable, prefijo `cor_`.
- `content.main` — datos del documento (contacto, fechas, divisa, líneas,
  totales, plantilla).
- `content.attachments` — adjuntos vinculados (ver [Adjuntos](#adjuntos)).
- `content.meta` — metadatos internos.

Dentro de `content.main` son obligatorios `docNumber` y `lines`. Campos con
comportamiento propio:

| Campo | Significado |
|---|---|
| `state` | Estado del pedido. Calculado; se ignora al escribir. |
| `owner` | Usuario responsable del pedido. Limita la visibilidad cuando se trabaja con roles personalizados. |
| `warehouse` | Almacén del documento a efectos de stock. Si no lo indicas se usa el almacén por defecto de la empresa. |
| `customFields` | Valores de campos personalizados del documento. |
| `counterpart` | Datos fiscales de la contraparte. |

En respuestas, además, en el nivel raíz: `tags`, `creationDate`,
`modificationDate` y `related`. **`related` solo transporta los objetos que
pidas con el parámetro `related`** (hoy, definiciones de impuestos): no
contiene los documentos en los que se haya convertido el pedido.

Si creas el pedido con OAuth y no envías `owner`, la API asigna como
responsable al usuario autenticado. Si usas una API key, queda sin responsable
salvo que envíes `owner` en el body.

### Líneas de detalle

`content.main.lines` es un array. Cada línea requiere `text`, `quantity` y
`unitPrice`. Además de los campos comunes a los documentos de venta
(`discount`/`discountRate`, `tax`, `document`, `account`, `lineTotal`,
`origin`), las líneas de pedido añaden:

- `id` — identidad estable de la línea (ver
  [Identidad de las líneas](#identidad-de-las-líneas)).
- `state` — estado de la línea (ver [Estado de línea](#estado-de-línea)).
- `provider` — ID del contacto proveedor que suministrará esa línea. Se usa
  para generar órdenes de compra a partir de líneas pendientes. Si la línea
  referencia un producto (`document`), ese producto debe tener activada la
  faceta de compra.
- `purchaseOrder`, `purchaseOrderLine` — vínculo con la orden de compra que
  abastece la línea. Los gestiona FacturaDirecta: no se pueden crear ni
  modificar desde el pedido.

Si omites `tax` en una línea, FacturaDirecta resuelve el impuesto por defecto a
partir del producto, del contacto y de la posición fiscal del documento.

Las líneas **con producto** deben llevar `quantity` mayor que cero: de esa
cantidad se derivan el stock comprometido y el aviso de bajo mínimo.

**Importes y totales.** Los totales del documento (`total`,
`totalBeforeTaxes`, `linesTotal`, `taxes`) se calculan a partir de las líneas.
Conviene dejarlos vacíos al crear o actualizar y leerlos de la respuesta.

## Enlace a la aplicación

Las respuestas que devuelven un pedido de cliente incluyen `webUrl` junto al resto de
los datos. En los listados, cada elemento de `items` lleva su propio enlace.

`webUrl` abre la ficha del elemento en la aplicación web de FacturaDirecta.
Puedes mostrarlo como enlace en tu integración sin construir rutas internas.
El navegador pedirá iniciar sesión con un usuario que tenga acceso a la
empresa. Es un campo opcional y no es un endpoint de la API: trátalo como una
URL para el usuario, no como una URL a la que enviar `$ACCESS_TOKEN`.

## Operaciones

- [Lista de pedidos](#lista-de-pedidos)
- [Crear pedido](#crear-pedido)
- [Obtener un pedido](#obtener-un-pedido)
- [Actualizar pedido](#actualizar-pedido)
- [Borrar pedido](#borrar-pedido)
- [Actualizar etiquetas](#actualizar-etiquetas)
- [Enviar pedido por correo](#enviar-pedido-por-correo)
- [Generar el pedido en PDF](#generar-el-pedido-en-pdf)
- [Adjuntos](#adjuntos)

## Lista de pedidos

`GET /{companyId}/clientOrders` devuelve los pedidos activos de la empresa,
paginados.

**Parámetros de consulta específicos:**

- **`state`** — estado del pedido: `pending`, `ready`, `delivered` o
  `canceled`. Repite el parámetro para incluir varios estados.
- **`minDate`** / **`maxDate`** — fecha del pedido (`YYYY-MM-DD`), inclusive.
- **`series`** — serie sin aplicar el formato de año (`##` o `####`).
- **`formattedSeries`** — serie ya formateada.
- **`minNumber`** / **`maxNumber`** — número secuencial, inclusive.
- **`contact`** — cliente del pedido. Admite varios valores.
- **`hasContact`** — `false` para obtener los pedidos sin contacto asociado.
- **`minTotal`** / **`maxTotal`** — importe total.
- **`currency`** — moneda (ISO 4217).
- **`country`** — país de la dirección de facturación (ISO 3166-1 alpha-2).
- **`emails`** — búsqueda parcial en los correos del documento, insensible a
  mayúsculas y acentos; todas las palabras deben coincidir.
- **`allTheseTags`** — el pedido debe llevar todas las etiquetas indicadas.
- **`anyOfTheseTags`** — basta con una de las etiquetas indicadas.
- **`hasTags`** — `true` devuelve solo pedidos con al menos una etiqueta y `false`, solo pedidos sin etiquetas.
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
  "https://app.facturadirecta.com/api/$COMPANY_ID/clientOrders?state=pending&state=ready&sortBy=-date&limit=50"
```

## Crear pedido

`POST /{companyId}/clientOrders` crea un pedido.

**Parámetros del body:**

- `content` (obligatorio) — el pedido. `content.type` debe ser `"clientOrder"`
  y `content.main` debe llevar `docNumber` y `lines`.
- `tags` (opcional) — etiquetas iniciales del pedido.

**Notas:**

- Si no envías `content.uuid`, la API asigna uno con prefijo `cor_`. Si lo
  envías y ya existe, responde `409 Conflict`.
- El `state` del pedido se calcula desde las líneas: lo que envíes en
  `main.state` se descarta.
- Toda referencia a otro documento (`main.contact`, `lines[].document`,
  `lines[].provider`, `main.paymentMethod`, `main.theme`) debe existir. Si no,
  la respuesta es `400` indicando los IDs no encontrados.

**Parámetros globales aceptados:** `accept-version`.

###### Ejemplo de request JSON

Contenido mínimo (heredado del ejemplo `minimal` del openapi). Las líneas se
crean en estado `pending`, así que el pedido queda `pending`:

```json
{
  "content": {
    "type": "clientOrder",
    "main": {
      "docNumber": { "series": "PED" },
      "contact": "con_2e3f4a18-9b7c-4d6a-8e1f-5c2d3b4a7e9f",
      "currency": "EUR",
      "lines": [
        {
          "quantity": 1,
          "unitPrice": 100,
          "tax": ["S_IVA_21"],
          "text": "Descripción del artículo pedido"
        }
      ]
    }
  }
}
```

Con estado y proveedor por línea (ejemplo `conEstadosDeLinea`). Todas las
líneas están `ready`, así que el pedido queda `ready`:

```json
{
  "content": {
    "type": "clientOrder",
    "main": {
      "docNumber": { "series": "PED" },
      "contact": "con_2e3f4a18-9b7c-4d6a-8e1f-5c2d3b4a7e9f",
      "currency": "EUR",
      "lines": [
        {
          "quantity": 2,
          "unitPrice": 50,
          "tax": ["S_IVA_21"],
          "text": "Artículo en stock listo para entregar",
          "state": "ready"
        },
        {
          "quantity": 1,
          "unitPrice": 75,
          "tax": ["S_IVA_21"],
          "text": "Artículo suministrado por proveedor",
          "state": "ready",
          "provider": "con_7b1d2c93-4e5f-4a6b-8c7d-9e0f1a2b3c4d"
        }
      ]
    }
  }
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '@clientOrder.json' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/clientOrders"
```

## Obtener un pedido

`GET /{companyId}/clientOrders/{id}` devuelve un pedido por ID.

**Parámetros de consulta específicos:**

- **`related`** — `taxIds` para recibir en `related.objects` las definiciones
  de los impuestos usados en el pedido.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/clientOrders/cor_1a2b3c44-5d6e-4f70-8192-a3b4c5d6e7f8?related=taxIds"
```

## Actualizar pedido

`PUT /{companyId}/clientOrders/{id}` sustituye **el contenido completo** del
pedido. No es un PATCH: lo que no envíes se pierde.

**Parámetros del body:**

- `content` (obligatorio) — el pedido completo. Si incluye `uuid`, debe
  coincidir con el de la ruta.
- `syncContactData` (opcional) — con `true`, actualiza dirección
  (`address`, `country`, `zipcode`) y `counterpart` a partir del contacto de
  `main.contact`. No toca correos, posición fiscal, vencimiento ni cuenta
  contable.
- `tags` y `tagsOperation` (opcionales) — etiquetas y operación a aplicar
  (`add`, `remove` o `replace`).

**Notas:**

- Conserva los `id` de las líneas que no quieras recrear (ver
  [Identidad de las líneas](#identidad-de-las-líneas)).
- No puedes eliminar, cambiar de producto, cambiar de cantidad ni cambiar de
  proveedor una línea vinculada a una orden de compra. Tampoco puedes crear
  el vínculo desde aquí: se crea desde la orden.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '@clientOrder.json' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/clientOrders/cor_1a2b3c44-5d6e-4f70-8192-a3b4c5d6e7f8"
```

## Borrar pedido

`DELETE /{companyId}/clientOrders/{id}` borra el pedido. El borrado es
recuperable desde la interfaz; el documento queda con `archived: true` y emite
el evento `client_order.archived`.

**Restricciones:**

- No puedes borrar un pedido con líneas vinculadas a órdenes de compra
  **activas**: la API responde `400` pidiendo que borres antes esas órdenes (o
  canceles sus líneas) para liberar los vínculos.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/clientOrders/cor_1a2b3c44-5d6e-4f70-8192-a3b4c5d6e7f8"
```

## Actualizar etiquetas

`PUT /{companyId}/clientOrders/{id}/tags` cambia solo las etiquetas, sin tocar
el contenido del pedido.

**Parámetros del body** (ambos obligatorios):

- `tags` — array de etiquetas.
- `tagsOperation` — `add`, `remove` o `replace`.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '{"tags":["urgente","q3-2026"],"tagsOperation":"add"}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/clientOrders/cor_1a2b3c44-5d6e-4f70-8192-a3b4c5d6e7f8/tags"
```

## Enviar pedido por correo

`PUT /{companyId}/clientOrders/{id}/send` envía el pedido por correo
electrónico. Los destinatarios, el asunto y el cuerpo se toman de la plantilla
del tema; los campos del body permiten sobreescribirlos puntualmente.

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
  -d '{"to":["compras@acme.es"],"subject":"Tu pedido PED-2026-14"}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/clientOrders/cor_1a2b3c44-5d6e-4f70-8192-a3b4c5d6e7f8/send"
```

## Generar el pedido en PDF

`PUT /{companyId}/clientOrders/{id}/pdf` genera una URL temporal con el pedido
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
  "https://app.facturadirecta.com/api/$COMPANY_ID/clientOrders/cor_1a2b3c44-5d6e-4f70-8192-a3b4c5d6e7f8/pdf"
```

## Adjuntos

Un pedido puede tener hasta **10 adjuntos**. Los adjuntos no se suben en el
body del pedido: primero se crean con `POST /uploads` y después se vinculan
enviando sus `uploadIds`.

### Listar adjuntos

`GET /{companyId}/clientOrders/{id}/attachments` devuelve los adjuntos
vinculados al pedido.

**Parámetros globales aceptados:** `accept-version`.

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/clientOrders/cor_1a2b3c44-5d6e-4f70-8192-a3b4c5d6e7f8/attachments"
```

### Vincular adjuntos

`POST /{companyId}/clientOrders/{id}/attachments` añade adjuntos al pedido a
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
  "https://app.facturadirecta.com/api/$COMPANY_ID/clientOrders/cor_1a2b3c44-5d6e-4f70-8192-a3b4c5d6e7f8/attachments"
```

### Eliminar un adjunto

`DELETE /{companyId}/clientOrders/{id}/attachments/{attachmentIndex}` elimina
un adjunto. `attachmentIndex` es su posición en el array, empezando en cero;
un valor no entero o negativo devuelve `400`.

**Parámetros globales aceptados:** `accept-version`.

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/clientOrders/cor_1a2b3c44-5d6e-4f70-8192-a3b4c5d6e7f8/attachments/0"
```

## Numeración

Los pedidos usan sus propias series de numeración, configurables en los ajustes
de la empresa. Si no indicas `docNumber.number` al crear, se asigna el
siguiente número de la serie. Las series con `##` o `####` se expanden con el
año en `docNumber.formattedSeries`.

## Webhooks

Los cambios en pedidos emiten `client_order.created`, `client_order.updated`,
`client_order.archived` y `client_order.unarchived`. Ver
[Webhooks](./webhooks.md).

Ten en cuenta que un pedido puede emitir `client_order.updated` **sin que tú lo
hayas tocado**: guardar o borrar una orden de compra vinculada propaga estados
a sus líneas. Ver [Flujo de pedidos y órdenes de compra](../guides/orders-flow.md).

## Errores comunes

- `400` — el documento tiene referencias que no existen (contacto, producto,
  proveedor, método de pago o plantilla).
- `400` — dos líneas con el mismo `id`, o un vínculo con orden de compra
  incompleto (documento vinculado y línea vinculada deben ir juntos).
- `400` — intento de crear o modificar desde el pedido el vínculo con una
  orden de compra: solo puede crearse desde la propia orden.
- `400` — eliminación, cambio de producto, de cantidad o de proveedor en una
  línea vinculada a una orden de compra.
- `400` — línea con producto y `quantity` menor o igual que cero.
- `400` — proveedor asignado a una línea cuyo producto no tiene activadas las
  preferencias de compra.
- `400` — borrado de un pedido con líneas vinculadas a órdenes de compra
  activas.
- `400` con código `attachments_limit_exceeded` — se supera el máximo de
  adjuntos del documento.
- `403` — el plan no incluye pedidos ni órdenes de compra (ver
  [Disponibilidad por plan](#disponibilidad-por-plan)), o faltan scopes.
- `404` — el pedido no existe en la empresa indicada.
- `409` — `content.uuid` ya existe al crear, o no hay destinatario ni
  remitente al enviar por correo.

Ver [Errores y validaciones](../guides/errors.md) para el formato general.

## Referencia exhaustiva

Esta página cubre los matices funcionales y los casos típicos. Para la
referencia exhaustiva de todos los campos del body y la respuesta, consulta el
[Swagger UI](https://www.facturadirecta.com/api) o el
[openapi crudo](https://app.facturadirecta.com/openapi.json).

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| GET | `/{companyId}/clientOrders` | `getClientOrders` | `clientOrders:read` | Lista de pedidos |
| GET | `/{companyId}/clientOrders/{id}` | `getClientOrder` | `clientOrders:read` | Obtener un pedido |
| GET | `/{companyId}/clientOrders/{id}/attachments` | `getClientOrderAttachments` | `clientOrders:read` | Listar adjuntos de un pedido |
| POST | `/{companyId}/clientOrders` | `createClientOrder` | `clientOrders:write` | Crear pedido |
| POST | `/{companyId}/clientOrders/{id}/attachments` | `addClientOrderAttachments` | `clientOrders:write` | Vincular adjuntos a un pedido |
| PUT | `/{companyId}/clientOrders/{id}` | `updateClientOrder` | `clientOrders:write` | Actualizar pedido |
| PUT | `/{companyId}/clientOrders/{id}/pdf` | `createClientOrderPDF` | `clientOrders:read` | Generar el pedido en formato PDF |
| PUT | `/{companyId}/clientOrders/{id}/send` | `sendClientOrder` | `clientOrders:write` | Enviar pedido por correo electrónico |
| PUT | `/{companyId}/clientOrders/{id}/tags` | `updateClientOrderTags` | `clientOrders:write` | Actualizar etiquetas de pedido |
| DELETE | `/{companyId}/clientOrders/{id}` | `deleteClientOrder` | `clientOrders:write` | Borrar pedido |
| DELETE | `/{companyId}/clientOrders/{id}/attachments/{attachmentIndex}` | `deleteClientOrderAttachment` | `clientOrders:write` | Eliminar adjunto de un pedido |

## Scopes

- **`clientOrders:read`** — Lectura de pedidos.
- **`clientOrders:write`** — Modificación de pedidos.
