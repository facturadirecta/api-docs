---
title: Idempotencia y reintentos seguros
audience:
  - developers
status: draft
---

# Idempotencia y reintentos seguros

Cuando una petición que modifica datos termina en timeout o se corta la
conexión, no sabes si el servidor llegó a ejecutarla. Reintentarla a ciegas
puede crear una factura duplicada o registrar dos veces el mismo cobro.

La cabecera `Idempotency-Key` resuelve ese caso: envías la misma clave en el
reintento y la API garantiza que la operación se ejecuta una sola vez. Si la
petición original ya terminó, recibes su respuesta tal cual, sin que se
ejecute nada.

Es opcional. Si no la envías, la API se comporta como siempre.

## Cómo se usa

Genera una clave única por cada operación que quieras hacer y envíala en la
cabecera. Si tienes que reintentar, repite **exactamente la misma petición con
la misma clave**.

```http
Idempotency-Key: 8e03978e-40d5-43e8-bc93-6894a57f9324
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -H "Idempotency-Key: 8e03978e-40d5-43e8-bc93-6894a57f9324" \
  -d '{
    "content": {
      "type": "contact",
      "main": {
        "name": "Cliente Empresa SL",
        "fiscalId": "B12345674",
        "currency": "EUR",
        "country": "ES",
        "address": "Pza Mayor, 4",
        "zipcode": "49004",
        "city": "Zamora",
        "accounts": { "client": "430000", "clientCredit": "438000" }
      }
    }
  }' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/contacts"
```

### Formato de la clave

- De **16 a 255 caracteres ASCII visibles, sin espacios**.
- Recomendamos un **UUID v4**. También vale un identificador con prefijo
  derivado de tu registro de origen, por ejemplo `migracion-factura-12345`, que
  es lo natural en una migración: la clave sale del dato y no tienes que
  guardarla.
- El mínimo de 16 caracteres evita colisiones accidentales. Si usaras el
  número de tu registro tal cual, la factura 123 y el contacto 123 tendrían la
  misma clave.
- **No pongas datos sensibles** en la clave: es una cabecera y queda
  registrada como tal.
- Si la envías entre comillas dobles (`"8e03978e-…"`), las comillas se
  ignoran: es la misma clave.

Una clave que no cumple el formato devuelve `400` con el código
`invalid_idempotency_key` y la operación no se ejecuta.

## Qué identifica una clave

- **Es única por empresa.** La misma clave en dos empresas distintas son dos
  claves distintas.
- **Dura 24 horas** desde la primera petición. Pasado ese plazo se trata como
  una clave nueva.
- **Queda ligada a la petición concreta**: método, ruta, parámetros de la URL y
  cuerpo. El orden de las propiedades del JSON y los espacios no cuentan: el
  mismo contenido serializado de otra forma es la misma petición.
- **La credencial no forma parte.** Si rotas tu API key entre la petición
  original y el reintento, la clave sigue funcionando. Por la misma razón,
  otra credencial de la misma empresa que envíe la misma clave con exactamente
  la misma petición recibe la respuesta guardada.

## Qué responde la API

| Situación | Respuesta |
|---|---|
| Primera petición con la clave | Se ejecuta con normalidad. Si termina bien, la respuesta se guarda 24 horas. |
| Misma clave y misma petición, con la original ya terminada | El mismo código y el mismo cuerpo que la original, sin ejecutar nada: ni cambios, ni webhooks, ni correos. |
| Misma clave, con la original todavía en curso | `409` con el código `idempotency_key_in_use` y la cabecera `Retry-After`. Espera esos segundos y reintenta. |
| Misma clave con otra petición (otro método, otra ruta u otro cuerpo) | `422` con el código `idempotency_key_reused`. Usa una clave nueva para cada operación distinta. |
| La petición original terminó en error | La clave queda libre y el reintento se ejecuta de nuevo. Única excepción: si el error se produjo cuando los datos ya estaban guardados, el reintento recibe la respuesta de la operación ya hecha, sin duplicarla. |

Que los errores no se guarden es deliberado. Si un alta falla por un dato
incorrecto, puedes corregirlo y reintentar **con la misma clave**, sin esperar
24 horas ni inventar otra. Es lo que necesitas cuando la clave sale de tu
registro de origen.

Los errores que se producen antes de ejecutar la operación (`401`, `403`,
`429` y los `400` de validación del cuerpo) tampoco consumen la clave.

### Cabeceras de la respuesta

| Cabecera | Cuándo aparece | Valor |
|---|---|---|
| `Idempotency-Key` | En toda respuesta a una petición con clave, en una operación que la admite | La clave aplicada |
| `Idempotent-Replayed` | Solo en respuestas repetidas | `true` |
| `Idempotent-Original-Date` | Solo en respuestas repetidas | Fecha de la ejecución original, en formato de fecha HTTP |
| `Idempotent-Original-Request-Id` | Solo en respuestas repetidas | El `X-Request-Id` de la petición original |

Respuesta a un reintento de una petición ya terminada:

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Idempotency-Key: 8e03978e-40d5-43e8-bc93-6894a57f9324
Idempotent-Replayed: true
Idempotent-Original-Date: Thu, 17 Sep 2026 10:15:30 GMT
Idempotent-Original-Request-Id: 5f0c2a1e-9b7d-4c3a-8e21-7d4f6a9b0c12
```

**Si la respuesta no trae la cabecera `Idempotency-Key`, la clave no se
aplicó**: esa operación no admite idempotencia y la cabecera se ignoró. Es la
forma de comprobarlo desde tu integración.

### Errores

El código de cada error va en `errors[0].code`, con la
[forma habitual de los errores](./errors.md):

```json
{
  "statusCode": 422,
  "message": "Esta Idempotency-Key ya se usó con otra petición (otro método, otra ruta u otro cuerpo). Usa una clave nueva para cada operación distinta.",
  "errors": [
    {
      "code": "idempotency_key_reused",
      "message": "Esta Idempotency-Key ya se usó con otra petición (otro método, otra ruta u otro cuerpo). Usa una clave nueva para cada operación distinta."
    }
  ]
}
```

| Código HTTP | `errors[0].code` | Qué hacer |
|---|---|---|
| `400` | `invalid_idempotency_key` | Corrige el formato de la clave. |
| `409` | `idempotency_key_in_use` | La original sigue en curso. Espera lo que indique `Retry-After` y reintenta con la misma clave. |
| `422` | `idempotency_key_reused` | Esa clave ya identifica otra petición. Genera una clave nueva. |
| `409` | `idempotency_result_not_replayable` | Ver «Operaciones que devuelven un secreto». |

## Operaciones que devuelven un secreto

`POST /{companyId}/apiKeys` y `POST /{companyId}/webhooks/endpoints` devuelven
un secreto que solo se muestra una vez (la API key y el secreto de firma) y
que FacturaDirecta no conserva en claro. La clave protege igual contra el
duplicado, pero la respuesta no se puede repetir.

Un reintento con la misma clave no crea nada y recibe un `409` que identifica
lo que se creó la primera vez:

```json
{
  "statusCode": 409,
  "message": "La petición original se completó, pero su respuesta contenía un secreto que no se conserva.",
  "errors": [
    {
      "code": "idempotency_result_not_replayable",
      "message": "La petición original se completó, pero su respuesta contenía un secreto que no se conserva.",
      "resource": { "type": "apiKey", "id": "<id-api-key>" }
    }
  ]
}
```

`resource.type` es `apiKey` o `webhookEndpoint`. Con ese `id` decides cómo
recuperar el secreto perdido:

- **API key**: bórrala con `DELETE /{companyId}/apiKeys/{id}` y crea otra con
  una clave de idempotencia nueva. Ver [API keys](../sections/api-keys.md).
- **Endpoint de webhook**: pide un secreto nuevo con
  `POST /{companyId}/webhooks/endpoints/{endpointId}/rotateSecret`, sin recrear
  el endpoint. Ver [Webhooks](../sections/webhooks.md).

## Operaciones que admiten la cabecera

La admiten **todas las operaciones `POST`, `PUT` y `DELETE`** salvo las de la
tabla siguiente. En la referencia de la API, las que la admiten declaran el
parámetro `Idempotency-Key`. Los `GET` la ignoran siempre.

| Operaciones que no la admiten | Por qué |
|---|---|
| `PUT /{companyId}/invoices/{id}/pdf` y el equivalente de facturas recurrentes, presupuestos, albaranes, pedidos y órdenes de compra; `PUT /{companyId}/invoices/{id}/facturae` | Son lecturas: generan un fichero temporal. Se pueden repetir sin riesgo. |
| `POST /{companyId}/inbox/{id}/proposeBill`, `proposeTicket` y `proposePayroll` | Son lecturas: devuelven un prototipo y no escriben nada. |
| `POST /{companyId}/webhooks/endpoints/{endpointId}/rotateSecret` | La respuesta lleva un secreto que no se conserva, y basta con volver a llamarla: el secreto válido es el de la última llamada. |
| `POST /{companyId}/uploads` y `POST /{companyId}/uploads/{uploadId}/commit` | Un upload duplicado es inocuo (caduca y se purga solo) y sellar dos veces el mismo upload da el mismo resultado. |
| `POST /{companyId}/inbox` | Tiene su propio mecanismo, que cubre otra necesidad. Ver el apartado siguiente. |

## `POST /inbox` es distinto

`POST /{companyId}/inbox` no admite la cabecera. Tiene un campo
`idempotencyKey` **en el cuerpo** que se llama parecido pero sirve para otra
cosa: evitar que el mismo documento entre dos veces en la bandeja de entrada.

| | Cabecera `Idempotency-Key` | Campo `idempotencyKey` de `POST /inbox` |
|---|---|---|
| Para qué sirve | Reintentar una petición tras un error de red | Que el mismo documento no se ingiera dos veces |
| Cuánto dura | 24 horas | No caduca mientras el item siga activo (sin archivar) |
| Compara la petición | Sí: otro cuerpo con la misma clave da `422` | No: con la misma clave devuelve el item existente, sea cual sea el fichero |
| Qué devuelve al repetir | La respuesta original, con `Idempotent-Replayed: true` | El item existente, con su estado actual |
| Clave habitual | Un UUID por operación | Una clave natural del documento, por ejemplo un hash del fichero |

Si tu proceso reenvía periódicamente los mismos documentos y confía en que la
bandeja no los duplique, usa el campo del cuerpo. Ver
[Bandeja de entrada](../sections/inbox.md).

## Relación con `content.uuid`

En las altas de documentos puedes proponer tú el identificador en
`content.uuid`. Un segundo alta con un `uuid` que ya existe en la empresa,
aunque el documento esté archivado, devuelve `409`. Sirve para no duplicar,
pero el reintento recibe un error en lugar del documento creado, y solo cubre
las altas.

`Idempotency-Key` es la forma recomendada de reintentar: cubre también
modificaciones, borrados, cobros y pagos, y el reintento recibe la respuesta
original. Las dos cosas son compatibles.

## Límite de peticiones

Una respuesta repetida **cuenta como una petición más** para el
[límite de peticiones](./authentication.md#límite-de-peticiones) de tu plan,
igual que un `409` por clave en uso.

Un `429` no consume la clave: el límite se comprueba antes de mirarla. Espera
lo que indique `Retry-After` y reintenta con la misma clave.

## Recomendaciones

- **Genera la clave antes del primer intento** y guárdala junto a tu registro,
  o derívala de él. Una clave nueva en cada reintento no protege de nada.
- **Reintenta con la misma petición exacta.** Si cambias el cuerpo, usa una
  clave nueva: con la misma recibirás `422`.
- **Ante un timeout o un corte de conexión**, reintenta con la misma clave con
  espera exponencial. Recibirás la respuesta original, o un `409` con
  `Retry-After` mientras la primera petición siga en curso.
- **Ante un `5xx`**, reintenta también con la misma clave. Si la operación no
  llegó a guardarse, se ejecuta de nuevo; si llegó a guardarse, recibes su
  respuesta sin duplicarla.
- **Comprueba el eco** `Idempotency-Key` en la respuesta si necesitas estar
  seguro de que la operación admite la cabecera.
- **No reutilices claves** entre operaciones distintas ni las recicles antes
  de 24 horas.
