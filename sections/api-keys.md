---
title: API keys
audience:
  - developers
status: draft
---

# API keys

Una API key es una credencial **por empresa** para autenticar
integraciones de máquina a máquina. Es el método alternativo a OAuth2
cuando tu integración solo necesita acceder a una empresa concreta y
no necesita actuar en nombre de un usuario humano. Ver
[Autenticación](../guides/authentication.md) para el contexto general.

Este recurso permite gestionar las API keys vía la propia API: crear,
listar, consultar y borrar. También permite consultar los scopes de la
clave que autentica la llamada actual.

> En los ejemplos de esta página, los IDs y prefijos son ilustrativos.
> Los reales se devuelven al crear cada API key.

## Cuatro principios duros del recurso

Estas reglas son distintas a las de cualquier otro recurso y conviene
tenerlas claras antes de operar:

1. **La clave completa solo se ve una vez**, en la respuesta de la
   creación. Después FacturaDirecta solo guarda un hash y el prefijo
   para identificación. Si la pierdes, hay que crear otra.
2. **No hay endpoint de modificación** (`PUT`). Las API keys no se
   actualizan: para cambiar nombre o scopes hay que crear una nueva y
   borrar la antigua. Esto evita reasignar scopes a una clave en uso
   por descuido.
3. **No se pueden borrar API keys creadas manualmente** desde la
   interfaz (`managedViaApi: false`). Solo las que tu integración ha
   creado por API (`managedViaApi: true`) son borrables por API. Es una
   protección contra que un cliente automático elimine accidentalmente
   las credenciales que un humano configuró.
4. **Una API key no puede borrarse a sí misma.** Si autenticas con la
   API key `K` e intentas `DELETE /apiKeys/K`, la API rechaza con `403`.

## Formato de la clave

Al crear una API key, la respuesta incluye un campo `apiKey` con la
clave **completa**, en formato:

```
<prefijo>.<sufijo>
```

- **Prefijo**: 6 caracteres alfanuméricos. Identifica la clave en logs,
  listas, y queda asociado a las llamadas que realiza. Es lo que ves
  después en el campo `prefix` de las respuestas.
- **Sufijo**: 32 caracteres alfanuméricos. Es el secreto real. Junto
  con el prefijo forman la clave única.

Ejemplo de clave válida:

```
aB3cD9.xY7zN2qK4mP8rT5sJ1hF6gL0vW3uE
```

Para usarla en peticiones a la API, se envía en el header
`facturadirecta-api-key` (sin `Bearer`, sin esquema). Ver
[Autenticación](../guides/authentication.md#api-key).

## Estructura

`PublicApiKeyInfo` (lo que devuelven los GETs):

| Campo | Tipo | Obligatorio | Significado |
|---|---|---|---|
| `id` | string | sí | Identificador único de la API key (UUID v4 sin prefijo). Se usa en los paths como `{id}`. |
| `name` | string | sí | Nombre descriptivo elegido al crear. |
| `prefix` | string | sí | Prefijo de 6 caracteres. |
| `scopes` | string[] | sí | Scopes autorizados de la clave. |
| `creationTime` | string ISO 8601 | sí | Fecha de creación. |
| `createdBy` | string | no | Usuario que creó la clave. |
| `managedViaApi` | boolean | sí | `true` si la clave fue creada vía API y por tanto es borrable vía API. `false` si se creó desde la interfaz. |

Al crear, además del `content: PublicApiKeyInfo`, la respuesta incluye
el campo `apiKey` con la clave completa.

## Operaciones

- [Información de la API key actual](#información-de-la-api-key-actual)
- [Lista de API keys](#lista-de-api-keys)
- [Crear API key](#crear-api-key)
- [Obtener una API key](#obtener-una-api-key)
- [Borrar una API key](#borrar-una-api-key)

## Información de la API key actual

`GET /{companyId}/apiKeyInfo` devuelve los scopes asociados a la
API key que autentica la petición actual. Úsalo cuando tu integración
necesita adaptar su comportamiento a los permisos reales de la clave
instalada en una empresa.

Este endpoint solo acepta autenticación mediante API key. Si llamas con
OAuth2, devuelve `401 Unauthorized` porque no hay una API key actual que
inspeccionar.

**Respuesta:**

```json
{
  "scopes": ["contacts:read", "contacts:write"]
}
```

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "facturadirecta-api-key: $API_KEY" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/apiKeyInfo"
```

## Lista de API keys

`GET /{companyId}/apiKeys` devuelve todas las API keys configuradas en
la empresa, sin datos sensibles (solo prefijos, no claves completas).

**Parámetros de consulta específicos:** ninguno.

**Parámetros globales aceptados:** `accept-version`.

**Notas:**

- La lista incluye **todas** las API keys de la empresa, las creadas
  vía API y las creadas manualmente. Distínguelas por `managedViaApi`.
- Sin paginación: devuelve el conjunto completo (ver patrón "sin
  paginación" en la [guía de paginación](../guides/pagination.md)).

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/apiKeys"
```

## Crear API key

`POST /{companyId}/apiKeys` crea una API key.

**Parámetros del body:**

- `name` (obligatorio) — nombre descriptivo para identificarla en la
  lista.
- `scopes` (obligatorio, mínimo 1) — lista de scopes que la nueva clave
  podrá usar. **Si el caller es una API key, los scopes deben ser
  subconjunto de los suyos**. Si el caller es OAuth, puede asignar
  cualquier scope disponible.

**Respuesta:**

```json
{
  "content": { "id": "...", "name": "...", "prefix": "...", "scopes": [...], ... },
  "apiKey": "<prefijo>.<sufijo>"
}
```

El campo `apiKey` **solo se ve aquí**. Guárdalo en seguro antes de
seguir; no hay forma de recuperarlo después.

**Parámetros globales aceptados:** `accept-version`.

###### Ejemplo de request JSON

```json
{
  "name": "Mi API key",
  "scopes": ["contacts:read"]
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"Mi API key","scopes":["contacts:read"]}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/apiKeys"
```

## Obtener una API key

`GET /{companyId}/apiKeys/{id}` devuelve los metadatos de una API key
por su ID.

No incluye la clave completa (solo el prefijo). Esa información ya no
existe en FacturaDirecta tras la creación.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/apiKeys/3f8b2c14-7d4e-4a9c-b6f1-2a5e8d3c4b7f"
```

## Borrar una API key

`DELETE /{companyId}/apiKeys/{id}` elimina la API key.

**Restricciones:**

- La API key objetivo debe tener `managedViaApi: true`. Si fue creada
  desde la interfaz, devuelve `403 Forbidden` con el mensaje "No se
  puede eliminar una API key que no fue creada vía API. Las API keys
  creadas manualmente solo pueden eliminarse desde la interfaz de
  usuario."
- Una API key no puede borrarse a sí misma. Si autenticas con la API
  key `K` e intentas borrarla, devuelve `403 Forbidden` con el mensaje
  "Una API key no puede eliminarse a sí misma".
- Si el `id` no corresponde a ninguna API key existente, devuelve
  `403 Forbidden` (no `404`).

**Respuesta:** `{ result: true }`.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/apiKeys/3f8b2c14-7d4e-4a9c-b6f1-2a5e8d3c4b7f"
```

## Patrón de rotación

Para rotar una API key sin interrumpir el servicio:

1. Crea una **nueva** API key con los mismos scopes que la antigua.
2. Actualiza tu integración para usar la nueva clave.
3. Verifica que la integración funciona con la nueva clave (deja un
   margen razonable).
4. Borra la antigua con `DELETE /apiKeys/{id}` (solo si fue creada vía
   API).

No hay "rotateSecret" como en webhooks: la API key es inmutable. La
rotación se hace siempre creando nueva y borrando antigua.

## Recomendaciones

- **Pide los scopes mínimos.** Si tu integración solo lee contactos,
  no pidas `contacts:write`. Es más seguro y más fácil de auditar.
- **Una API key por integración, no compartas.** Si una integración
  deja de usarse, borra su API key.
- **Guarda la clave (`<prefijo>.<sufijo>`) en un gestor de secretos**,
  no en código fuente ni en logs.
- **Las API keys son por empresa.** Para multi-tenant SaaS, OAuth2 con
  `offline_access` es el patrón correcto (no API keys una por empresa).
- **Para auto-aprovisionamiento de claves desde tu sistema**, usa
  OAuth2 para autenticar tu sistema admin y crear API keys para cada
  cliente. Sigue siendo más seguro que distribuir API keys
  hardcodeadas.

## Errores comunes

- `400 ValidationError` — `scopes` vacío o `name` ausente.
- `403 Forbidden` — el `id` no existe, o `managedViaApi: false`, o
  intento de auto-eliminación.
- `401 Unauthorized` — llamada a `GET /apiKeyInfo` autenticada con
  OAuth2 en lugar de API key.
- `403 Forbidden` — scope solicitado fuera del subconjunto del caller
  (cuando el caller autentica con API key).

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
| GET | `/{companyId}/apiKeyInfo` | `getPublicApiKeyInfo` | — | Información de la API key actual |
| GET | `/{companyId}/apiKeys` | `listPublicApiKeys` | `apiKeys:read` | Lista de API keys |
| GET | `/{companyId}/apiKeys/{id}` | `getPublicApiKey` | `apiKeys:read` | Obtener API key |
| POST | `/{companyId}/apiKeys` | `createPublicApiKey` | `apiKeys:write` | Crear API key |
| DELETE | `/{companyId}/apiKeys/{id}` | `deletePublicApiKey` | `apiKeys:write` | Eliminar API key |

## Scopes

- **`apiKeys:read`** — Lectura de API keys.
- **`apiKeys:write`** — Gestión de API keys (creación y eliminación).
