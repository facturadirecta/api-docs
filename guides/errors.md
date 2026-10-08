---
title: Errores y validaciones
audience:
  - developers
status: draft
---

# Errores y validaciones

Las respuestas de error de la API pública siguen siempre la misma forma
base, independientemente del recurso. Esta guía documenta los códigos HTTP
que verás, el formato JSON, y cómo distinguir entre los distintos tipos.

## Forma general de la respuesta

Toda respuesta de error tiene como mínimo:

```json
{
  "statusCode": 400,
  "message": "Mensaje legible que explica el problema"
}
```

Puede incluir además:

- **`type`** — categoría del error cuando aplica.
- **`errors`** — array con detalles por campo en errores de validación
  (solo en `400 ValidationError`).
- **`code`** — motivo estable de algunos errores de autorización.
- **`requiredScope`** — scope necesario para completar la operación.
- **`companyId`** — empresa en la que se ha denegado el acceso.
- **`manageUrl`** — página donde el usuario puede ampliar una aplicación
  conectada, cuando esa acción resuelve el error.

`Content-Type` siempre es `application/json`.

## Códigos HTTP

| Código | Significado típico |
|---|---|
| `400 Bad Request` | El cuerpo o los parámetros no cumplen el contrato. Si es un fallo de validación de campos, viene con `errors[]` (ver más abajo). |
| `401 Unauthorized` | Falta credencial, está expirada o es inválida. Ver [Autenticación](./authentication.md). |
| `403 Forbidden` | Credencial válida pero sin los scopes necesarios, o sin acceso a la empresa indicada en el path. |
| `404 Not Found` | El recurso no existe o no pertenece a la empresa del path. |
| `409 Conflict` | La operación choca con el estado actual: identificador duplicado, borrado de un recurso con dependencias, etc. También lo devuelve una petición con `Idempotency-Key` cuando la original sigue en curso. Ver [Idempotencia](./idempotency.md). |
| `422 Unprocessable Entity` | La `Idempotency-Key` enviada ya se usó con otra petición distinta. Ver [Idempotencia](./idempotency.md). También lo devuelve una consulta de la [actividad](../sections/activity.md) que tarda demasiado (`activity_query_timeout`): acótala con `minDate` y `maxDate`. Y un [informe](../sections/reports.md) que tarda demasiado (`report_query_timeout`): acota el periodo, el nivel de detalle o las columnas. |
| `429 Too Many Requests` | Demasiadas escrituras simultáneas del mismo tipo en la empresa. Reintenta con backoff. Ver [Límite de peticiones](./authentication.md#límite-de-peticiones). |
| `500 Internal Server Error` | Error inesperado del servidor. No es un error del cliente; conviene reintentar tras un retraso. |

## Errores de validación (`400`)

Cuando el body o los parámetros fallan el JSON Schema, la respuesta incluye
un array `errors` con un objeto por cada problema:

```json
{
  "statusCode": 400,
  "message": "Validation failed",
  "type": "ValidationError",
  "errors": [
    {
      "path": "content.main.fiscalId",
      "message": "Formato de NIF/CIF inválido para España"
    },
    {
      "path": "content.main.lines.0.unitPrice",
      "message": "Debe ser un número mayor que 0"
    }
  ]
}
```

Cada entrada del array localiza el campo problemático y explica qué falla.
Para integraciones que muestran el error al usuario final, basta con
mostrar el `message` de cada entrada; el `path` ayuda al developer a
depurar.

## Fechas que no existen (`400`)

Los filtros de fecha de los listados tienen que ser una fecha del
calendario. Una fecha con la forma correcta que no existe, como
`2026-02-30`, `2026-13-01` o `2025-02-29`, responde `400` con un
`message` que nombra el parámetro. El filtro nunca se ignora ni se
sustituye por otro valor.

```json
{
  "statusCode": 400,
  "message": "'minDate' no es una fecha válida (AAAA-MM-DD)"
}
```

Qué admite cada filtro depende del campo que filtra:

- **Filtros por día**, como la fecha de un documento: solo admiten
  `AAAA-MM-DD`, sin hora. Son `minDate` y `maxDate` de
  [facturas](../sections/invoices.md), [presupuestos](../sections/estimates.md),
  [pedidos](../sections/client-orders.md), [órdenes de compra](../sections/purchase-orders.md),
  [albaranes](../sections/delivery-notes.md), [nóminas](../sections/payrolls.md),
  [facturas de compra](../sections/bills.md), [movimientos bancarios](../sections/statements.md)
  y [diario](../sections/journal.md). Una fecha que no existe responde con el
  mensaje anterior.
- **Filtros por instante**, como la fecha de creación o la de un registro
  de actividad: admiten fecha y hora ISO 8601 o solo la fecha, que cuenta
  el día entero. Son los filtros comunes `minCreationDate`,
  `maxCreationDate`, `minModificationDate` y `maxModificationDate` (ver
  [Paginación](./pagination.md#filtros-estándar-de-fecha)), `minDate` y
  `maxDate` de la [bandeja de entrada](../sections/inbox.md) y de la
  [actividad](../sections/activity.md), y `minNextScheduledDate` y
  `maxNextScheduledDate` de [facturas recurrentes](../sections/recurring.md). Con
  solo la fecha (`2026-02-30`), el mensaje es el anterior; con fecha y
  hora (`2026-02-30T10:00:00.000Z`), es
  `Valor incorrecto para 'minCreationDate': debe ser una fecha y hora en formato ISO 8601`.

El mismo nombre puede ser de los dos tipos: `minDate` de las facturas es
un día, y el de la bandeja de entrada, un instante. La página de cada
recurso indica cuál es.

Una fecha con otra forma, como `2026-2-3` o `03/02/2026`, también responde
`400`.

## Errores de negocio (`400`, `409`)

Algunos errores no son fallos de schema sino reglas de negocio:

- Crear una factura con una serie ya consumida.
- Borrar un contacto que tiene documentos asociados.
- Modificar un presupuesto convertido en factura.

Estos errores vienen como `400` o `409` (según el caso) con un `message`
descriptivo. Cuando el mensaje incluye datos relevantes (ID de documento
bloqueante, valor duplicado), se devuelven en campos adicionales del JSON.

## Errores de autenticación y autorización (`401`, `403`)

- **`401`** — el token o la API key son rechazados antes de evaluar
  permisos. Casos típicos: token expirado, API key revocada, header mal
  formado.
- **`403`** — la credencial es válida pero le falta algo:
  - Scopes insuficientes para la operación (por ejemplo, intentar `POST` con scope `read`).
  - La empresa del path no es accesible para esta credencial.
  - El rol del usuario en la empresa no permite la operación. Con OAuth se
    aplican los scopes del token y los permisos del rol del usuario en esa
    empresa, también si es un rol personalizado.

El `403` por permisos indica el scope en `requiredScope` y el motivo en
`code`:

```json
{
  "statusCode": 403,
  "message": "No tienes permisos para realizar esta operación",
  "requiredScope": "invoices:write",
  "code": "user_role_insufficient",
  "companyId": "com_…"
}
```

| `code` | Motivo | Qué hacer |
| --- | --- | --- |
| `token_scope_missing` | El token o la API key no incluye el scope. | Volver a autorizar pidiendo el scope, o usar una API key que lo tenga. |
| `user_role_insufficient` | El usuario no pertenece a la empresa o su rol no lo permite. El nivel de rol «solo lo asignado» no da acceso por la API. | Pedir el permiso a un administrador de la empresa: volver a autorizar no lo arregla. |
| `plan_api_not_available` | El plan de la empresa no incluye el acceso por la API. | Cambiar de plan. |
| `connection_permission_insufficient` | La aplicación conectada no tiene el permiso necesario en esa empresa. | Abrir `manageUrl`, ampliar el permiso y repetir la llamada. |
| `company_not_in_connection` | La empresa no está autorizada para esa aplicación conectada. | Abrir `manageUrl`, añadir la empresa y repetir la llamada. |
| `connection_required` | El cliente MCP o CLI exige una conexión, pero la petición no la presenta. | Volver a conectar la aplicación o iniciar sesión de nuevo en el CLI. |
| `connection_revoked` | La conexión no existe, se ha desconectado o no corresponde al usuario del token. | Iniciar una conexión nueva. |

`companyId` solo aparece si el usuario pertenece a la empresa del path.
`manageUrl` aparece cuando el acceso se puede ampliar desde **Aplicaciones
conectadas**. Usa `code`, no el texto, para distinguir los casos.

Un contacto puede tener varias facetas. Si una escritura afecta, por ejemplo,
a un contacto que también es proveedor, la credencial necesita permiso sobre
todas las facetas implicadas. El `403` usa
`connection_permission_insufficient` o `user_role_insufficient` y explica el
permiso que falta.

Para detalles del flujo de auth y rotación de tokens, ver
[Autenticación](./authentication.md).

## Límites del plan

Cuando una operación no puede completarse porque la empresa supera una
capacidad del plan contratado, la respuesta incluye el código estable
`plan_limit_exceeded` dentro de `errors`:

```json
{
  "statusCode": 403,
  "message": "La empresa está en modo solo lectura por exceder el límite de clientes contratados (10). Por favor, amplia tu suscripción o reduce el número de clientes de tu empresa.",
  "errors": [
    {
      "message": "La empresa está en modo solo lectura por exceder el límite de clientes contratados (10). Por favor, amplia tu suscripción o reduce el número de clientes de tu empresa.",
      "code": "plan_limit_exceeded"
    }
  ]
}
```

El mismo código se devuelve al crear una factura con el cupo del plan agotado
(por año natural: 100 en Gratis, 2.000 en Bronce, 5.000 en Plata y 12.000 en
Oro; sin cupo en Diamante). Solo bloquea crear facturas: el resto de
operaciones sigue disponible y la ampliación es subir de plan.

Se comportan igual otras capacidades: número de empleados, productos, productos
con control de stock, campos personalizados, almacenes y **documentos
recurrentes activos**. En todos los casos, el mensaje indica el tope alcanzado.

El estado HTTP es siempre `403`. Usa `errors[].code`, no el texto por separado,
para identificar este caso. No reintentes la misma operación hasta ampliar el
plan o reducir el uso que supera la capacidad.

### Funcionalidades no incluidas en el plan

El mismo código `plan_limit_exceeded` identifica el `403` de una funcionalidad
entera que el plan no incluye: las escrituras de [pedidos](../sections/client-orders.md)
y [órdenes de compra](../sections/purchase-orders.md) («Los pedidos de cliente y las
órdenes de compra no están disponibles en tu plan») y las operaciones de stock
de [productos](../sections/products.md) («El control de stock no está disponible en tu
plan»). La diferencia con un tope de capacidad está solo en el mensaje: aquí no
hay uso que reducir, y la salida es un plan o un módulo que incluya la
funcionalidad.

FacturaDirecta también envía un e-mail de aviso al propietario de la empresa,
al usuario que hizo la solicitud —o al creador de la API key— y a los usuarios
con permiso completo de configuración de empresa. El aviso identifica la
petición y el motivo. Se envía como máximo una vez por empresa cada 7 días,
aunque el error se repita.

### Informes no incluidos en el plan

Los [informes contables](../sections/reports.md) (pérdidas y ganancias, balance de
situación y sumas y saldos) necesitan el módulo de Contabilidad. Sin él
responden `403` con `plan_limit_exceeded` y la funcionalidad que falta en
`hint`:

```json
{
  "statusCode": 403,
  "message": "Este informe no está disponible en tu plan",
  "errors": [
    {
      "message": "Este informe no está disponible en tu plan",
      "code": "plan_limit_exceeded",
      "hint": { "features": ["fullAccounting"] }
    }
  ]
}
```

Este error no envía el e-mail de aviso a los administradores. Para saber de
antemano si una empresa tiene el módulo, una aplicación conectada puede leer
`accountingModule` en el [perfil](../sections/profile.md). El
[resumen de resultados](../sections/reports.md) está disponible en todos los planes.

## Errores de idempotencia (`400`, `409`, `422`)

Las peticiones con cabecera `Idempotency-Key` pueden devolver cuatro códigos
estables en `errors[0].code`:

| Código HTTP | `errors[0].code` | Significado |
|---|---|---|
| `400` | `invalid_idempotency_key` | La clave no cumple el formato (de 16 a 255 caracteres ASCII visibles, sin espacios). |
| `409` | `idempotency_key_in_use` | Hay otra petición en curso con la misma clave. La respuesta trae `Retry-After`. |
| `422` | `idempotency_key_reused` | La clave ya se usó con otra petición: otro método, otra ruta u otro cuerpo. |
| `409` | `idempotency_result_not_replayable` | La petición original se completó, pero su respuesta llevaba un secreto que no se conserva. `errors[0].resource` identifica lo que se creó. |

El detalle de cada caso y qué hacer está en [Idempotencia](./idempotency.md).

## Recursos no encontrados (`404`)

`404` significa siempre **una de estas dos cosas**:

- El ID no existe en la empresa indicada.
- El ID existe pero pertenece a otra empresa (la API no revela que el
  recurso existe en otro tenant).

Trata ambos casos como "no existe para mí" en tu integración.

## Estrategia recomendada de manejo

- **Diferencia errores del cliente (4xx) de errores del servidor (5xx).**
  Los 4xx no deben reintentarse sin cambiar el input; los 5xx sí, con
  backoff exponencial.
- **Para `400 ValidationError`, recorre `errors[]`** y muestra al
  usuario los `message` específicos en vez del `message` general.
- **Reintentos seguros.** Envía la cabecera `Idempotency-Key` en las
  operaciones que modifican datos: si reintentas con la misma clave tras un
  timeout o un corte de conexión, la operación no se ejecuta dos veces y
  recibes la respuesta original. Ver [Idempotencia](./idempotency.md).
- **No deduzcas estado a partir del código HTTP solo.** Combina código y
  contenido (`type`, `message`, `errors[].code`) para clasificar el error en tu
  lógica.

## Ejemplo completo

Petición que viola dos validaciones (CIF mal formado y línea con precio
negativo):

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "statusCode": 400,
  "message": "Validation failed",
  "type": "ValidationError",
  "errors": [
    {
      "path": "content.main.fiscalId",
      "message": "Formato de NIF/CIF inválido para España"
    },
    {
      "path": "content.main.lines.0.unitPrice",
      "message": "Debe ser un número mayor que 0"
    }
  ]
}
```

Y conflicto al borrar un contacto con factura asociada:

```http
HTTP/1.1 409 Conflict
Content-Type: application/json

{
  "statusCode": 409,
  "message": "No se puede borrar el contacto: tiene 3 facturas asociadas"
}
```
