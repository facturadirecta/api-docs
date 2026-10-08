---
title: Perfil de usuario
audience:
  - developers
status: draft
---

# Perfil de usuario

El recurso `profile` devuelve los datos básicos del **usuario
autenticado** y las **empresas disponibles para la credencial actual**. Es
la operación que usa una integración OAuth para resolver qué `companyId` puede
usar en el resto de llamadas.

Cuando la petición procede de una aplicación conectada, la respuesta incluye
solo las empresas que el usuario autorizó para esa conexión y añade sus permisos
efectivos.

## Excepciones de este endpoint

`/profile` es el único endpoint de la API pública que rompe con dos
convenciones del resto del catálogo:

- **No lleva `companyId` en la ruta.** El path es literalmente
  `/profile`: el usuario puede tener acceso a varias empresas, y este
  endpoint sirve precisamente para descubrirlas antes de fijar una.
- **Es OAuth-only.** Las [API Keys](./api-keys.md) **no pueden
  llamar** a este endpoint: una API key está ya ligada a una empresa
  concreta, así que no tiene sentido pedir su perfil de empresas.

## Scope

El scope efectivo es `profile`. No aparece como permiso
seleccionable porque **se incluye automáticamente** en el cliente
OAuth `facturadirecta-api` al hacer login. No hace falta pedirlo
explícitamente.

## Estructura

El endpoint devuelve `{ profile }` con la siguiente forma:

| Campo | Tipo | Significado |
|---|---|---|
| `username` | string | Email del usuario autenticado (es el identificador único). |
| `email` | string | Mismo valor que `username`. Se mantiene por compatibilidad. |
| `firstName` | string | Nombre del usuario (del campo `given_name` del JWT). |
| `lastName` | string | Apellidos del usuario (del campo `family_name` del JWT). |
| `companies` | array | Empresas disponibles para la credencial actual. |
| `connection` | object | Solo con una aplicación conectada. Identifica la conexión y permite abrir su gestión. |

El objeto opcional `connection` contiene:

| Campo | Tipo | Significado |
|---|---|---|
| `id` | string | Identificador de la conexión. |
| `clientName` | string | Nombre de la aplicación conectada. |
| `manageUrl` | string | Página de FacturaDirecta donde el usuario puede cambiar sus empresas y permisos. |

Cada elemento de `companies`:

| Campo | Tipo | Significado |
|---|---|---|
| `id` | string | ID de empresa. Es el valor que debes usar como `{companyId}` en las llamadas al resto de la API. |
| `name` | string | Nombre de la empresa. Si la empresa tiene primer apellido (`surname`), aparece concatenado con un espacio. |
| `taxCode` | string | CIF/NIF de la empresa. |
| `brand` | string | Marca comercial, si está informada (opcional). |
| `owner` | string | Email del propietario de la empresa. |
| `role` | string | Rol del usuario autenticado en esa empresa concreta. |
| `permissions` | object | Solo con una aplicación conectada. Permisos efectivos por recurso: `read` para lectura y `full` para lectura y escritura. |
| `accountingModule` | boolean | Solo con una aplicación conectada. `true` si la empresa tiene el módulo de Contabilidad, contratado o incluido en su plan o en la prueba. Sin él no están disponibles los informes contables: pérdidas y ganancias, balance de situación y sumas y saldos. |
| `manageUrl` | string | Solo con una aplicación conectada. Página para cambiar los permisos de esa empresa. |

## Operaciones

- [Obtener el perfil](#obtener-el-perfil)

## Obtener el perfil

`GET /profile` devuelve el perfil del usuario autenticado y sus
empresas.

**No lleva parámetros de path ni de consulta.** Solo el header
`accept-version` para fijar la versión de la API. Ver
[Autenticación](../guides/authentication.md).

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/profile"
```

Respuesta típica:

```json
{
  "profile": {
    "username": "ana@ejemplo.com",
    "email": "ana@ejemplo.com",
    "firstName": "Ana",
    "lastName": "García López",
    "companies": [
      {
        "id": "com_3a7e8d29-4f5b-4c6e-9a1d-2b8c4e7f5d3a",
        "name": "Estudio Diseño SL",
        "taxCode": "B12345674",
        "owner": "ana@ejemplo.com",
        "role": "admin"
      },
      {
        "id": "com_8b1c5e94-2d6a-4f7e-9c3b-1a5d8f4e2c7b",
        "name": "Cooperativa Norte",
        "taxCode": "F87654328",
        "brand": "Norte",
        "owner": "luis@cooperativa.com",
        "role": "accountant"
      }
    ]
  }
}
```

### Respuesta de una aplicación conectada

Una aplicación conectada recibe solo la selección autorizada. Los permisos ya
están limitados por la concesión, el rol actual del usuario y el plan de cada
empresa:

```json
{
  "profile": {
    "username": "ana@ejemplo.com",
    "email": "ana@ejemplo.com",
    "firstName": "Ana",
    "lastName": "García López",
    "companies": [
      {
        "id": "com_3a7e8d29-4f5b-4c6e-9a1d-2b8c4e7f5d3a",
        "name": "Estudio Diseño SL",
        "taxCode": "B12345674",
        "owner": "ana@ejemplo.com",
        "role": "admin",
        "permissions": {
          "contacts": "read",
          "invoices": "full"
        },
        "accountingModule": false,
        "manageUrl": "https://app.facturadirecta.com/connections/<id-conexion>?company=<id-empresa>"
      }
    ],
    "connection": {
      "id": "<id-conexion>",
      "clientName": "Asistente de ejemplo",
      "manageUrl": "https://app.facturadirecta.com/connections/<id-conexion>"
    }
  }
}
```

## Flujo típico de integración

1. El usuario completa el login OAuth. Tu integración recibe el
   `access_token`.
2. Llamas a `GET /profile` para descubrir las empresas disponibles.
3. Si aparece `connection`, respetas la selección y los `permissions` de cada
   empresa. Los `manageUrl` permiten que el usuario amplíe el acceso sin volver
   a conectar.
4. Eliges una empresa entre las devueltas.
5. Usas su `id` como `{companyId}` en las llamadas siguientes a la API.

## Recomendaciones

- **No cachees indefinidamente**: la lista de empresas y sus permisos pueden
  cambiar cuando el usuario modifica la conexión, cambia de rol o deja una
  empresa. Refresca al iniciar sesión y ante un `401` o `403`.
- **`role` informa, no autoriza por sí solo**: la autorización efectiva combina
  los scopes de la credencial y los permisos del rol. En una aplicación
  conectada también se limita por la concesión de cada empresa.
- **Usa `permissions` cuando esté presente** para adaptar las acciones de tu
  aplicación antes de intentar una escritura.
- **Usa `accountingModule` para elegir el informe** antes de pedirlo: sin el
  módulo de Contabilidad, la empresa no tiene los informes contables.
- **Para API Keys, no llames a `/profile`**: la API key ya conoce su
  empresa. Si tu integración alterna entre OAuth y API Key,
  detéctalo en el cliente y omite el paso.

## Errores comunes

- `401 Unauthorized` (`code: "auth_*"`) — el `access_token` no es
  válido o ha expirado. Refresca el token vía OAuth.
- `403 Forbidden` — intento de llamar con una API key. Este endpoint
  es OAuth-only por diseño.
- Una empresa que no pertenece a la conexión no aparece en `companies`. Si se
  intenta usar directamente, el resto de endpoints responde `403` con
  `code: "company_not_in_connection"` y puede incluir `manageUrl`.

Ver [Errores y validaciones](../guides/errors.md) y
[Autenticación](../guides/authentication.md) para los detalles del
flujo OAuth.

## Referencia exhaustiva

Esta página cubre el único endpoint del recurso y todos sus campos.
Para el contrato JSON literal, consulta el
[Swagger UI](https://www.facturadirecta.com/api) o el
[openapi crudo](https://app.facturadirecta.com/openapi.json).

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| GET | `/profile` | `getProfile` | — | Perfil de usuario |
