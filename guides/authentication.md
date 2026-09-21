---
title: Autenticación
audience:
  - developers
status: draft
---

# Autenticación

La API pública de FacturaDirecta soporta dos métodos de autenticación,
[OAuth2](#oauth2) y [API key](#api-key). Cada operación acepta ambos; tú
eliges el que mejor encaje con tu integración. Ambos comparten los mismos
scopes y operan sobre el mismo conjunto de endpoints.

## Endpoint base

Todas las URLs de la API pública en producción tienen el prefijo:

```
https://app.facturadirecta.com/api
```

A partir de ahí, los paths que se documentan en cada recurso (por ejemplo
`/{companyId}/contacts`) se concatenan a esta base.

Recursos auxiliares:

- **Referencia OpenAPI navegable (Swagger UI):** `https://www.facturadirecta.com/api`
- **Especificación OpenAPI cruda (`openapi.json`):** `https://app.facturadirecta.com/openapi.json`

## OAuth2

Recomendado cuando un usuario conecta su propia empresa de FacturaDirecta a
tu aplicación. Tú obtienes un token en su nombre y operas dentro de su
empresa.

FacturaDirecta usa Keycloak como servicio de autorización, con realm
`facturadirecta`. El flujo es `authorizationCode` estándar:

1. Rediriges al usuario a la URL de autorización con `client_id=facturadirecta-api`
   y los scopes que necesitas.
2. El usuario autentica y autoriza los scopes.
3. Keycloak redirige a tu callback con un `code`.
4. Intercambias `code` por `access_token` (y opcionalmente `refresh_token`)
   en el endpoint de token.
5. Usas el `access_token` en el header `Authorization: Bearer <token>` para
   cada llamada a la API.

**URLs del servicio de autorización en producción:**

| Endpoint | URL |
|---|---|
| Autorización | `https://auth.facturadirecta.com/auth/realms/facturadirecta/protocol/openid-connect/auth` |
| Token | `https://auth.facturadirecta.com/auth/realms/facturadirecta/protocol/openid-connect/token` |
| Refresh | `https://auth.facturadirecta.com/auth/realms/facturadirecta/protocol/openid-connect/token` |

La fuente autoritativa es el `openapi.json` público en
`https://app.facturadirecta.com/openapi.json`, sección
`components.securitySchemes.oAuth.flows.authorizationCode`.

**`client_id`** para integraciones externas es `facturadirecta-api` y
**no requiere `client_secret`**.

La API pública solo acepta tokens emitidos para clientes OAuth autorizados
por FacturaDirecta. Los clientes gestionados por FacturaDirecta para
integraciones propias, como conectores MCP autorizados, pueden usar su
propio `client_id`, pero siguen sujetos a los scopes del token. Un token
válido del realm no basta si el cliente OAuth que lo emitió no está
autorizado para la API pública.

**`offline_access`** como scope te permite obtener un `refresh_token` para
operar sin nueva intervención del usuario.

### Ejemplo de cabecera

```http
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIi...
```

## API key

Si tu integración solo necesita acceder a una empresa concreta que tú
administras, puedes generar una API key desde **Ajustes > Integraciones**
en la propia empresa de FacturaDirecta, o crearla por API desde
`POST /apiKeys`.

Al crear una API key le asignas los scopes que va a poder usar (los mismos
que están disponibles para OAuth2, salvo `offline_access` que para una API
key no tiene sentido).

### Ejemplo de cabecera

```http
facturadirecta-api-key: aB3cD9.xY7zN2qK4mP8rT5sJ1hF6gL0vW3uE
```

Formato: `<prefijo de 6 caracteres>.<sufijo de 32 caracteres>`. Es lo que
devuelve `POST /apiKeys` en el campo `apiKey` de la respuesta (solo
visible en la creación; tras eso, FacturaDirecta solo conserva el hash).
La API key se envía como header propio, **no** como `Authorization: Bearer`.

## Versión de la API

Todas las peticiones aceptan el header `accept-version` con el número de
versión de la API. Si no lo envías, se usa la versión vigente definida por el
servidor. Para producción se recomienda fijarlo explícitamente:

```http
accept-version: 1.0.10
```

El servidor solo aplica versiones que se hayan publicado. Si envías una
versión desconocida o con un formato no válido, aplica la versión vigente. La
cabecera `X-Api-Version` de la respuesta indica qué versión se ha aplicado.

## Scopes

Cada operación de la API declara los scopes que requiere. Los scopes siguen
el patrón `<recurso>:<acción>`:

- `<recurso>:read` — permite leer (`GET`).
- `<recurso>:write` — permite crear, modificar y borrar (`POST`, `PUT`, `DELETE`).

El conjunto exacto de scopes y su descripción está en el openapi público y
en cada página de recurso de esta documentación. Pide siempre los scopes
mínimos necesarios para la integración.

Dos scopes se salen del patrón y conviene conocerlos:

- **`banks:readIban`** — scope adicional que añade el IBAN completo a las
  consultas de bancos y de métodos de pago. Ver [Bancos](../sections/banks.md).
- **`files:write`** — permite subir archivos para adjuntarlos a documentos.
  Ver [Uploads](../sections/uploads.md).

Los documentos del **Módulo Inventario** tienen sus propios scopes:
`clientOrders:read` y `clientOrders:write` para
[pedidos](../sections/client-orders.md), y `purchaseOrders:read` y
`purchaseOrders:write` para [órdenes de compra](../sections/purchase-orders.md).
Tener el scope no basta: si el plan de la empresa no incluye esos
documentos, las escrituras responden `403`. Las operaciones de stock de
productos siguen usando `products:read` y `products:write`, y también
dependen de que el plan incluya el control de stock.

## Límite de peticiones

Cada empresa tiene un **límite de peticiones por minuto** que depende de su
plan. El límite se cuenta por empresa (no por credencial) en ventanas de un
minuto naturales, y solo se aplica al tráfico de integración: llamadas con API
key y con clientes OAuth de integración.

| Plan | Peticiones por minuto |
|---|---|
| Gratis | 30 |
| Bronce | 60 |
| Plata | 120 |
| Oro | 300 |
| Diamante | 600 |

Esos son los valores de la familia de planes vigente. Algunas cuentas con
planes anteriores o con condiciones particulares tienen otro límite: no lo
supongas a partir del plan, léelo de la cabecera de la respuesta.

Cada respuesta incluye estas cabeceras:

```http
X-RateLimit-Limit: 120
X-RateLimit-Remaining: 118
X-RateLimit-Reset: 1789012860
```

- **`X-RateLimit-Limit`** — peticiones permitidas en la ventana actual.
- **`X-RateLimit-Remaining`** — peticiones que te quedan en esta ventana.
- **`X-RateLimit-Reset`** — timestamp Unix (segundos) en que empieza la ventana
  siguiente.

Cuando una petición supera el límite, la respuesta añade además
**`Retry-After`** con los segundos que faltan para la ventana siguiente. Hoy
esa petición **se sigue atendiendo**: el exceso queda registrado pero no se
rechaza. Aun así, diseña la integración para respetar el límite —lee
`X-RateLimit-Remaining` y espacia las llamadas— y para tratar un
`429 Too Many Requests` con reintento tras `Retry-After`.

## Errores de autenticación

- **`401 Unauthorized`** — token expirado, ausente, mal formado, emitido
  por un cliente OAuth no autorizado para la API pública, o API key inválida.
  Refresca el token (si tienes `refresh_token`), revisa el `client_id` o
  regenera la API key.
- **`403 Forbidden`** — el token o la API key son válidos, pero les falta
  algún scope para la operación. Revisa los scopes asignados.
- **`403 Forbidden` con `companyId` ajeno** — el token no tiene acceso a
  la empresa indicada en el path. En OAuth, el usuario solo accede a
  empresas en las que tiene rol; en API key, solo a la empresa en la que
  se creó.

Para el formato general de errores ver [Errores y validaciones](./errors.md).

## Recomendaciones

- **No registres tokens ni API keys en logs**, ni siquiera durante
  desarrollo.
- **Limita los scopes** al mínimo necesario. Si solo lees contactos, no
  pidas `contacts:write`.
- **Una API key por integración.** Si una integración deja de usarse,
  rota la API key antes de borrarla.
- **Para multi-tenant SaaS**, OAuth2 con `offline_access` es el patrón
  correcto. API keys son por empresa y no escalan a "una clave para
  varias empresas".
