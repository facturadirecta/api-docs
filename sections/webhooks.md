---
title: Webhooks
audience:
  - developers
status: draft
---

# Webhooks

Los webhooks notifican en tiempo real cuando ocurren cambios en los
recursos de tu empresa. En vez de hacer polling, configuras una URL
HTTPS y FacturaDirecta envía una petición `POST` con el detalle del
cambio cada vez que se produce un evento al que estás suscrito.

Esta página es la **referencia técnica** para desarrolladores: payloads, firma,
eventos, endpoints de API y reintentos. Si lo que necesitas es crear o gestionar
un endpoint desde la interfaz de FacturaDirecta, consulta
[Configurar webhooks](https://help.facturadirecta.com/es/articles/15097369-configurar-webhooks).

<!-- source: legacy-13905258 -->

Son útiles para integraciones con ERPs, CRMs, plataformas de e-commerce,
asesorías y cualquier automatización que necesite reaccionar a cambios en la
cuenta de facturación sin hacer polling. Antes de configurarlos en producción,
pruébalos en un [entorno sandbox](https://help.facturadirecta.com/es/articles/15157082-entorno-de-pruebas-sandbox).

Los webhooks están disponibles en los planes **Avanzado** y **Total**.

> En los ejemplos de esta página, los UUIDs (`whe_…`, `con_…`, `inv_…`)
> y los `signing_secret` son **ilustrativos**. Cada endpoint tiene los
> suyos; sustitúyelos por los valores reales que devuelve la API.

## Dos sub-recursos

El recurso `webhooks` se divide en dos partes:

- **Endpoints** (`/webhooks/endpoints/...`) — la configuración: URLs
  destino, eventos a los que se suscriben, activación, rotación del
  secret de firma.
- **Eventos** (`/webhooks/events/...`) — la **historia de entregas**: cada
  cambio en un recurso genera un evento por cada endpoint suscrito.
  Permite inspeccionar qué se envió, su estado y reintentar entregas
  fallidas.

Cada endpoint se suscribe a uno o más tipos de evento. Formato:
`<recurso>.<acción>`. La lista completa de tipos disponibles está
más abajo:

## Catálogo de eventos

Eventos disponibles para suscripción en `events` al crear o
actualizar un endpoint:

| Recurso | Eventos |
|---|---|
| Facturas | `invoice.created`, `invoice.updated`, `invoice.archived`, `invoice.unarchived`, `invoice.sent`, `invoice.voided`, `invoice.verifactu_sent` |
| Gastos | `expense.created`, `expense.updated`, `expense.archived`, `expense.unarchived` |
| Contactos | `contact.created`, `contact.updated`, `contact.archived`, `contact.unarchived` |
| Presupuestos | `estimate.created`, `estimate.updated`, `estimate.archived`, `estimate.unarchived` |
| Albaranes | `delivery_note.created`, `delivery_note.updated`, `delivery_note.archived`, `delivery_note.unarchived` |
| Transacciones | `transaction.created`, `transaction.updated`, `transaction.archived`, `transaction.unarchived` |
| Productos | `product.created`, `product.updated`, `product.archived`, `product.unarchived` |
| VeriFactu | `verifactu_batch.sent` |
| Bandeja de entrada | `inbox.scanned` |

El catálogo puede ampliarse ante cambios; no asumas un conjunto cerrado.
Para suscribirte a todos los eventos de una categoría, declara cada tipo
explícitamente.

## Entornos sandbox y producción

<!-- source: webhookDispatcher -->

Los endpoints mantienen una separación total entre entornos:

- Un endpoint creado en una empresa sandbox recibe eventos del sandbox y sus
  payloads llevan `livemode: false`.
- Un endpoint creado en una empresa real recibe eventos de producción y sus
  payloads llevan `livemode: true`.

Un endpoint de sandbox no recibe eventos de producción, y un endpoint de
producción no recibe eventos de sandbox.

## Forma del payload entregado

Cuando se dispara un evento, FacturaDirecta envía un `POST` a la URL del
endpoint con este cuerpo JSON:

```json
{
  "id": "whe_5a8b1c34-2d4e-4f6a-8b9c-1e3f5a7b9c0d",
  "type": "invoice.created",
  "object_type": "invoice",
  "created": 1715616000,
  "livemode": true,
  "company_id": "com_2c4f8e21-7b3a-4d9c-9e1f-a8b7c6d5e4f3",
  "data": {
    "object": { "...": "snapshot del recurso afectado" },
    "previous_attributes": { "...": "diff con el estado anterior (solo en eventos *.updated)" }
  }
}
```

Notas:

- `id` (prefijo `whe_`) identifica la entrega. Es el mismo valor que el
  `id` del evento en `GET /webhooks/events/{id}`.
- `type` coincide con uno de los tipos del catálogo.
- `object_type` es el tipo de recurso afectado (`invoice`, `contact`...).
- `created` es timestamp Unix en segundos.
- `livemode: true` indica que es un evento real; `false` corresponde a
  eventos simulados (sandbox o reintentos manuales).
- `data.object` contiene el snapshot completo del recurso en el momento
  del evento.
- `data.previous_attributes` aparece **solo en eventos `*.updated`** y
  contiene un diff con los valores anteriores de los campos modificados.

## Firma del payload

Cada petición incluye dos cabeceras HTTP para que verifiques que el
payload viene realmente de FacturaDirecta:

```http
Webhook-Signature: t=1715616000,v1=8f3d4e7c2b9a1f5e6d0c8b7a3e5f9d1c2b4a6e8f0d3c5b7a9e1f3d5c7b9a0e2f
Webhook-Timestamp: 1715616000
```

Algoritmo, equivalente al de Stripe:

1. Toma el `Webhook-Timestamp` y el cuerpo crudo de la petición.
2. Concatena: `signed_payload = "<timestamp>.<raw_body>"`.
3. Calcula `HMAC-SHA256(secret, signed_payload)` y compáralo con el valor
   `v1=...` del header `Webhook-Signature`.
4. Rechaza la petición si la diferencia entre `Webhook-Timestamp` y la
   hora actual supera **5 minutos** (anti-replay).

El `secret` es el `signing_secret` devuelto al crear el endpoint
(formato `whsec_<hex>`) o tras `rotateSecret`. **Solo se muestra una vez**
en la creación/rotación. Guárdalo en un lugar seguro al recibirlo.

## Política de reintentos

Si tu endpoint responde con un código distinto de `2xx` o no responde,
FacturaDirecta reintenta la entrega con backoff exponencial. Calendario
hardcoded:

| Intento | Espera desde el anterior | Tiempo total desde el primer envío |
|---|---|---|
| 1 | — | 0 |
| 2 | 15 segundos | 15 s |
| 3 | 45 segundos | 1 min |
| 4 | 2 minutos | 3 min |
| 5 | 5 minutos | 8 min |
| 6 | 15 minutos | 23 min |
| 7 | 45 minutos | 1 h 8 min |
| 8 | 2 horas | 3 h 8 min |
| 9 | 5 horas | 8 h 8 min |
| 10 | 10 horas | 18 h 8 min |
| 11 | 18 horas | 36 h 8 min |
| 12 | 36 horas | ~72 horas |

Tras 12 intentos sin éxito el evento queda en estado `failed`. Puedes
reintentar manualmente con `POST /webhooks/events/{id}/retry`.

## Operaciones

- [Endpoints](#endpoints)
  - [Crear endpoint](#crear-endpoint)
  - [Listar endpoints](#listar-endpoints)
  - [Obtener un endpoint](#obtener-un-endpoint)
  - [Actualizar endpoint](#actualizar-endpoint)
  - [Borrar endpoint](#borrar-endpoint)
  - [Activar endpoint](#activar-endpoint)
  - [Desactivar endpoint](#desactivar-endpoint)
  - [Rotar secret de firma](#rotar-secret-de-firma)
- [Eventos](#eventos)
  - [Listar eventos](#listar-eventos)
  - [Obtener un evento](#obtener-un-evento)
  - [Reintentar un evento](#reintentar-un-evento)

## Endpoints

### Crear endpoint

`POST /{companyId}/webhooks/endpoints` crea un endpoint con su lista
inicial de suscripciones.

**Parámetros del body:**

- `name` (obligatorio) — nombre descriptivo del endpoint.
- `url` (obligatorio) — URL HTTPS de destino.
- `events` (obligatorio) — array de tipos de evento del [catálogo](#catálogo-de-eventos).

**Respuesta:** `{ content: <endpoint>, signing_secret: "whsec_<hex>" }`.
El `signing_secret` **solo se muestra en la creación**; guárdalo en
seguro.

###### Ejemplo de request JSON

```json
{
  "name": "Integración ERP — facturas",
  "url": "https://api.miempresa.com/webhooks/facturadirecta",
  "events": ["invoice.created", "invoice.updated", "invoice.voided"]
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{
    "name": "Integración ERP — facturas",
    "url": "https://api.miempresa.com/webhooks/facturadirecta",
    "events": ["invoice.created", "invoice.updated", "invoice.voided"]
  }' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/webhooks/endpoints"
```

### Listar endpoints

`GET /{companyId}/webhooks/endpoints` devuelve los endpoints configurados
en la empresa.

**Respuesta:** `{ items: WebhookEndpoint[] }`. Sin paginación (ver patrón
sin paginación en la [guía de paginación](../guides/pagination.md)).

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/webhooks/endpoints"
```

### Obtener un endpoint

`GET /{companyId}/webhooks/endpoints/{endpointId}` devuelve un endpoint
por ID.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/webhooks/endpoints/whe_5a8b1c34-2d4e-4f6a-8b9c-1e3f5a7b9c0d"
```

### Actualizar endpoint

`PUT /{companyId}/webhooks/endpoints/{endpointId}` aplica una
**actualización parcial**: los campos omitidos conservan su valor actual.
A diferencia de los recursos documento, **no es un reemplazo completo**.

**Parámetros del body** (todos opcionales):

- `name`
- `url`
- `events` (la lista nueva reemplaza la anterior).

###### Ejemplo de request JSON

Cambiar solo la lista de eventos a los que se suscribe:

```json
{
  "events": ["invoice.created", "invoice.voided"]
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '{"events":["invoice.created","invoice.voided"]}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/webhooks/endpoints/whe_5a8b1c34-2d4e-4f6a-8b9c-1e3f5a7b9c0d"
```

### Borrar endpoint

`DELETE /{companyId}/webhooks/endpoints/{endpointId}` elimina el endpoint.
Las entregas pendientes y el historial de eventos asociados quedan
disponibles en `/webhooks/events` para consulta, pero **no se reintentan**
tras el borrado.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/webhooks/endpoints/whe_5a8b1c34-2d4e-4f6a-8b9c-1e3f5a7b9c0d"
```

### Activar endpoint

`POST /{companyId}/webhooks/endpoints/{endpointId}/enable` reactiva un
endpoint previamente desactivado. Solo afecta a eventos futuros; no
reintenta entregas pasadas (para eso, ver [Reintentar un evento](#reintentar-un-evento)).

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  -X POST \
  "https://app.facturadirecta.com/api/$COMPANY_ID/webhooks/endpoints/whe_5a8b1c34-2d4e-4f6a-8b9c-1e3f5a7b9c0d/enable"
```

### Desactivar endpoint

`POST /{companyId}/webhooks/endpoints/{endpointId}/disable` deja al
endpoint sin recibir eventos nuevos. Los eventos pendientes en cola
**no se enviarán**.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  -X POST \
  "https://app.facturadirecta.com/api/$COMPANY_ID/webhooks/endpoints/whe_5a8b1c34-2d4e-4f6a-8b9c-1e3f5a7b9c0d/disable"
```

### Rotar secret de firma

`POST /{companyId}/webhooks/endpoints/{endpointId}/rotateSecret` genera
un nuevo `signing_secret` para el endpoint e invalida el anterior.

**Respuesta:** `{ signing_secret: "whsec_<hex>" }`. **Solo se muestra una
vez**; guárdalo en seguro antes de actualizar tu integración.

Estrategia recomendada: rotar en horario de baja actividad. Mientras
actualizas tu integración, los eventos firmados con el secret antiguo
fallarán hasta que verifiques con el nuevo.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  -X POST \
  "https://app.facturadirecta.com/api/$COMPANY_ID/webhooks/endpoints/whe_5a8b1c34-2d4e-4f6a-8b9c-1e3f5a7b9c0d/rotateSecret"
```

## Eventos

### Listar eventos

`GET /{companyId}/webhooks/events` devuelve el historial de entregas con
**paginación por cursor** (ver el patrón B en la [guía de paginación](../guides/pagination.md)).

**Parámetros de consulta:**

- `endpointId` — filtrar por endpoint.
- `type` — filtrar por tipo de evento (p. ej. `invoice.created`).
- `status` — filtrar por estado: `pending`, `delivered`, `failed`.
- `limit` — máximo `100`.
- `cursor` — `created_at` del último evento de la página anterior, en
  formato ISO 8601 UTC. Omitir para la primera página.

**Respuesta:** `{ items: WebhookEvent[], hasMore: boolean }`. El cursor
para la siguiente página es el `created_at` del último elemento.

###### Copy as cURL

Primera página:

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/webhooks/events?status=failed&limit=100"
```

Siguiente página:

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/webhooks/events?status=failed&limit=100&cursor=2026-04-15T10:22:31.000Z"
```

### Obtener un evento

`GET /{companyId}/webhooks/events/{eventId}` devuelve un evento por ID
con su detalle (incluyendo el payload enviado y el resultado del último
intento).

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/webhooks/events/whe_5a8b1c34-2d4e-4f6a-8b9c-1e3f5a7b9c0d"
```

### Reintentar un evento

`POST /{companyId}/webhooks/events/{eventId}/retry` encola una nueva
entrega del evento contra el endpoint configurado.

Útil para:

- Eventos en estado `failed` cuando ya has solucionado el problema en
  tu integración.
- Eventos `delivered` cuando necesitas re-procesarlos en tu sistema (la
  API no impide reintentar entregas exitosas).

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  -X POST \
  "https://app.facturadirecta.com/api/$COMPANY_ID/webhooks/events/whe_5a8b1c34-2d4e-4f6a-8b9c-1e3f5a7b9c0d/retry"
```

## Recomendaciones para integradores

- **Verifica siempre la firma** antes de procesar el payload. No confíes
  en el remitente del header `Host` ni en certificados TLS; el secret
  HMAC es la única garantía.
- **Responde rápido (2xx)** para evitar reintentos. Si necesitas
  procesamiento largo, encola el evento en tu lado y responde `200`
  inmediatamente.
- **Asume entregas duplicadas.** Tu integración debe ser idempotente: el
  `id` del evento es estable y sirve para deduplicar.
- **Trata `event.id` como llave primaria.** El campo `id` (`whe_...`) es
  único por entrega y persiste entre reintentos.
- **Para sincronizaciones desde cero**, en vez de paginar todo el
  historial guarda el `created_at` del último evento procesado con éxito
  y reanuda desde ahí.
- **Tras rotar el secret**, soporta brevemente ambos secrets (antiguo y
  nuevo) hasta que confirmes que todos los eventos en vuelo se firmaron
  con el nuevo.

## Errores comunes

- `400 ValidationError` — `url` no es HTTPS o `events` contiene un tipo
  desconocido.
- `404 Not Found` — el `endpointId` o `eventId` no existe en la empresa.
- `409 Conflict` — intento de rotar secret en un endpoint borrado.

Ver [Errores y validaciones](../guides/errors.md) para el formato general.

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| GET | `/{companyId}/webhooks/endpoints` | `listPublicWebhookEndpoints` | `webhooks:read` | Lista de endpoints de webhooks |
| GET | `/{companyId}/webhooks/endpoints/{endpointId}` | `getPublicWebhookEndpoint` | `webhooks:read` | Obtener endpoint de webhook |
| GET | `/{companyId}/webhooks/events` | `listPublicWebhookEvents` | `webhooks:read` | Lista de eventos de webhook |
| GET | `/{companyId}/webhooks/events/{eventId}` | `getPublicWebhookEvent` | `webhooks:read` | Obtener evento de webhook |
| POST | `/{companyId}/webhooks/endpoints` | `createPublicWebhookEndpoint` | `webhooks:write` | Crear endpoint de webhook |
| POST | `/{companyId}/webhooks/endpoints/{endpointId}/disable` | `disablePublicWebhookEndpoint` | `webhooks:write` | Desactivar endpoint de webhook |
| POST | `/{companyId}/webhooks/endpoints/{endpointId}/enable` | `enablePublicWebhookEndpoint` | `webhooks:write` | Activar endpoint de webhook |
| POST | `/{companyId}/webhooks/endpoints/{endpointId}/rotateSecret` | `rotatePublicWebhookEndpointSecret` | `webhooks:write` | Rotar secret de firma del endpoint |
| POST | `/{companyId}/webhooks/events/{eventId}/retry` | `retryPublicWebhookEvent` | `webhooks:write` | Reintentar entrega de evento |
| PUT | `/{companyId}/webhooks/endpoints/{endpointId}` | `updatePublicWebhookEndpoint` | `webhooks:write` | Actualizar endpoint de webhook |
| DELETE | `/{companyId}/webhooks/endpoints/{endpointId}` | `deletePublicWebhookEndpoint` | `webhooks:write` | Eliminar endpoint de webhook |

## Scopes

- **`webhooks:read`** — Leer configuración y historial de webhooks.
- **`webhooks:write`** — Gestionar endpoints de webhooks.
