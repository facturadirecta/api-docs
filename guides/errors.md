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

`Content-Type` siempre es `application/json`.

## Códigos HTTP

| Código | Significado típico |
|---|---|
| `400 Bad Request` | El cuerpo o los parámetros no cumplen el contrato. Si es un fallo de validación de campos, viene con `errors[]` (ver más abajo). |
| `401 Unauthorized` | Falta credencial, está expirada o es inválida. Ver [Autenticación](./authentication.md). |
| `403 Forbidden` | Credencial válida pero sin los scopes necesarios, o sin acceso a la empresa indicada en el path. |
| `404 Not Found` | El recurso no existe o no pertenece a la empresa del path. |
| `409 Conflict` | La operación choca con el estado actual: identificador duplicado, borrado de un recurso con dependencias, etc. |
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

Para detalles del flujo de auth y rotación de tokens, ver
[Autenticación](./authentication.md).

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
- **Idempotencia.** Repetir un `POST /contacts` con el mismo
  `fiscalId` da `409 Conflict`, no duplicado. Aprovecha esto para
  reintentos seguros en sincronizaciones.
- **No deduzcas estado a partir del código HTTP solo.** Combina código y
  contenido (`type`, `message`) para clasificar el error en tu lógica.

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
