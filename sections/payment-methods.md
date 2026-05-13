---
title: Métodos de pago
audience:
  - developers
status: draft
---

# Métodos de pago

Un método de pago describe **cómo se cobra o se paga** un documento.
Puede ser una descripción manual ("Pago por transferencia") que solo
se muestra al cliente o al proveedor, o un instrumento SEPA con datos
bancarios reales (mandato de domiciliación, IBAN para transferencia).

Los métodos de pago se asocian a documentos (facturas, presupuestos,
gastos, ...) en su campo `paymentMethod` y determinan qué se imprime
en la plantilla y cómo se procesa el cobro/pago en operaciones de
banca.

> En los ejemplos de esta página:
>
> - Los **UUIDs** (`pam_…`, `ban_…`, `con_…`) son ilustrativos. Cada
>   empresa tiene los suyos; sustitúyelos por los identificadores
>   reales que devuelve la API.
> - Los **IBAN** son ficticios. Usa siempre IBANs reales (con dígito
>   de control válido) en producción.

## Globales vs por contacto

Un método de pago puede ser:

- **Global de la empresa** (sin `contact`): aparece como opción por
  defecto en todos los documentos. Útil para "Pago por transferencia
  estándar" que vale para cualquier cliente o proveedor.
- **Por contacto** (con `contact`): solo aplica al cliente o proveedor
  indicado. Típico en mandatos SEPA, donde cada cliente firma su
  propio mandato de domiciliación.

El filtro `includeGlobal` en la lista permite incluir o excluir los
globales del resultado.

## Tipos: manual y SEPA

`content.main.subtype` determina la forma del método:

- **`manual`** — solo descripción textual. El sistema **no** procesa
  cobros/pagos automáticos: es informativo para el documento. No se
  permite `contact`.
- **`sepaDirectDebit`** — mandato para **cobros domiciliados** (recibos
  SEPA). El contacto es obligatorio. Lleva ID de mandato, fecha de
  firma, esquema (B2B o CORE) e IBAN del deudor.
- **`sepaCreditTransfer`** — **mandato para pagos por transferencia**
  SEPA. El contacto es obligatorio. Lleva IBAN del beneficiario.

La estructura de `content.main.details` cambia según el `subtype` (un
`oneOf` en el schema). Los detalles relevantes están en
[Estructura](#estructura).

## Dirección: send vs receive

`content.main.direction`:

- **`send`** — método para **hacer pagos** (a proveedores, normalmente).
- **`receive`** — método para **recibir cobros** (de clientes,
  normalmente).

En métodos manuales suele omitirse. En SEPA es coherente con el
`subtype`: `sepaDirectDebit` casi siempre es `receive` (cobras del
cliente), `sepaCreditTransfer` casi siempre es `send` (pagas al
proveedor).

## Estructura

- `content.type` — siempre `"paymentMethod"`.
- `content.uuid` — identificador inmutable, prefijo `pam_`.
- `content.main`:
  - `title` (obligatorio) — nombre del método en los listados.
  - `subtype` (obligatorio) — ver [Tipos](#tipos-manual-y-sepa).
  - `direction` — `send` o `receive`.
  - `contact` (obligatorio en SEPA, prohibido en manual) — ID del
    contacto asociado.
  - `bank` (opcional, solo en manual) — ID del banco propio asociado.
  - `details` (obligatorio) — varía según el `subtype`.

### Detalles según `subtype`

**`subtype: "manual"`**: `details` es un objeto vacío `{}`.

**`subtype: "sepaDirectDebit"`**: `details` contiene:

| Campo | Tipo | Obligatorio | Significado |
|---|---|---|---|
| `id` | string | sí | Identificador del mandato. |
| `signatureDate` | date | sí | Fecha de firma del mandato (puede ser `null`). |
| `instrument` | `"B2B"` \| `"CORE"` | sí | Esquema de adeudo SEPA. CORE es el estándar B2C; B2B es entre empresas. |
| `recurrent` | boolean | sí | Si el mandato cubre cobros recurrentes (`true`) o un único cobro (`false`). |
| `iban` | string | sí | IBAN de la cuenta del deudor. |
| `bic` | string | no | Código BIC/SWIFT (solo necesario en cobros internacionales). |
| `iban4` | string | sí (lo calcula la API) | Últimos 4 dígitos del IBAN. |

**`subtype: "sepaCreditTransfer"`**: `details` contiene:

| Campo | Tipo | Obligatorio | Significado |
|---|---|---|---|
| `iban` | string | sí | IBAN de la cuenta del beneficiario. |
| `bic` | string | no | Código BIC/SWIFT (solo necesario internacional). |
| `iban4` | string | sí (lo calcula la API) | Últimos 4 dígitos del IBAN. |

**Nota sobre `iban4`**: el schema lo declara como requerido en
respuesta, pero al crear se calcula automáticamente a partir del
`iban`. No es necesario enviarlo en el body.

## Operaciones

- [Lista de métodos de pago](#lista-de-métodos-de-pago)
- [Crear método de pago](#crear-método-de-pago)
- [Obtener un método de pago](#obtener-un-método-de-pago)
- [Actualizar método de pago](#actualizar-método-de-pago)
- [Borrar método de pago](#borrar-método-de-pago)

## Lista de métodos de pago

`GET /{companyId}/paymentMethods` devuelve los métodos de pago de la
empresa, paginados.

**Parámetros de consulta específicos:**

- **`contact`** — ID de contacto. Si se indica, solo devuelve métodos
  asociados a ese contacto.
- **`includeGlobal`** — `true`/`false` para incluir métodos globales de
  la empresa (sin contacto) en el resultado.
- **`sortBy`** — campo de orden.

**Parámetros globales aceptados:**

Acepta además los parámetros estándar `offset`, `limit`,
`minCreationDate`, `maxCreationDate`, `minModificationDate`,
`maxModificationDate` y el header `accept-version`. Ver
[Paginación](../guides/pagination.md) y
[Autenticación](../guides/authentication.md).

**Notas:**

- Para ver el IBAN completo (no solo `iban4`) en la respuesta, la
  credencial debe tener el scope `paymentMethods:readIban`. Sin ese
  scope la API solo devuelve `iban4`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/paymentMethods?contact=con_2c4f8e21-7b3a-4d9c-9e1f-a8b7c6d5e4f3&includeGlobal=true"
```

## Crear método de pago

`POST /{companyId}/paymentMethods` crea un método de pago.

**Parámetros del body:**

- `content.type` — siempre `"paymentMethod"`.
- `content.main.title` — obligatorio.
- `content.main.subtype` — obligatorio (`manual`, `sepaDirectDebit`,
  `sepaCreditTransfer`).
- `content.main.contact` — obligatorio en `sepa*`, prohibido en
  `manual`. Si se omite en `manual`, el método es global de la empresa.
- `content.main.details` — obligatorio. Estructura según `subtype`
  (ver [Detalles según subtype](#detalles-según-subtype)).
- `tags` (opcional).

**Parámetros globales aceptados:** `accept-version`.

###### Ejemplo de request JSON

Método manual global (heredado del ejemplo `transfer` del openapi):

```json
{
  "content": {
    "type": "paymentMethod",
    "main": {
      "subtype": "manual",
      "title": "Pago por transferencia",
      "bank": "ban_5e7d8a31-9c4b-4f6e-a1d3-2b5c7e9f1a4d",
      "details": {}
    }
  }
}
```

Mandato SEPA de domiciliación para un cliente (heredado del ejemplo
`directDebit`):

```json
{
  "content": {
    "type": "paymentMethod",
    "main": {
      "subtype": "sepaDirectDebit",
      "title": "Recibo domiciliado",
      "direction": "receive",
      "contact": "con_2c4f8e21-7b3a-4d9c-9e1f-a8b7c6d5e4f3",
      "details": {
        "id": "MANDATO_01",
        "signatureDate": "2023-04-10",
        "instrument": "CORE",
        "recurrent": true,
        "iban": "ES3011112222003333333333"
      }
    }
  }
}
```

Mandato de transferencia SEPA para un proveedor (heredado del ejemplo
`sepaCreditTransfer`):

```json
{
  "content": {
    "type": "paymentMethod",
    "main": {
      "subtype": "sepaCreditTransfer",
      "title": "Transferencia SEPA",
      "direction": "send",
      "contact": "con_9d4e2a73-1f6c-4b8d-a2e5-3b7c1d4e6f8a",
      "details": {
        "iban": "ES9122223333304444444444"
      }
    }
  }
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '@paymentMethod.json' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/paymentMethods"
```

## Obtener un método de pago

`GET /{companyId}/paymentMethods/{id}` devuelve un método de pago por
ID.

**Parámetros globales aceptados:** `accept-version`.

**Notas:**

- Sin scope `paymentMethods:readIban`, la respuesta enmascara el IBAN
  completo (solo se devuelve `iban4`).

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/paymentMethods/pam_4b7c1e29-2d8f-4a5e-b9c3-6d8a1e4f2c7b"
```

## Actualizar método de pago

`PUT /{companyId}/paymentMethods/{id}` sustituye **el contenido
completo** del método. No es un PATCH.

**Notas:**

- No se puede cambiar el `subtype` de un método existente. Para
  cambiarlo, crea uno nuevo y borra el anterior.
- Cambiar el IBAN de un mandato SEPA puede afectar a cobros/pagos
  pendientes vinculados.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '@paymentMethod.json' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/paymentMethods/pam_4b7c1e29-2d8f-4a5e-b9c3-6d8a1e4f2c7b"
```

## Borrar método de pago

`DELETE /{companyId}/paymentMethods/{id}` elimina el método de pago.

**Restricciones:**

- Si el método está referenciado por documentos existentes, la API
  rechaza el borrado con `409 Conflict`. Para "retirar" un método ya
  usado, créa uno nuevo y deja el antiguo sin asignar.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/paymentMethods/pam_4b7c1e29-2d8f-4a5e-b9c3-6d8a1e4f2c7b"
```

## Scopes especiales: `paymentMethods:readIban`

Por defecto, las respuestas que incluyen un método de pago SEPA
muestran solo `iban4` (los últimos 4 dígitos) por privacidad. Para
recibir el `iban` completo:

- Pide el scope **`paymentMethods:readIban`** en tu token OAuth o API
  key.
- También existe el scope **`banks:readIban`** para ver el IBAN
  completo de los bancos propios en las consultas de bancos.

Sin esos scopes, las operaciones funcionan pero los IBAN aparecen
truncados.

## Recomendaciones

- **Crea métodos globales** para las formas de pago estándar de la
  empresa ("Pago al contado", "Transferencia bancaria"). Aparecen como
  opciones por defecto en todos los documentos.
- **Crea métodos por contacto** solo cuando hay datos específicos del
  cliente o proveedor (típicamente mandatos SEPA con su IBAN).
- **No mezcles `subtype: manual` con `contact`**: la API rechaza esa
  combinación.
- **Pide `paymentMethods:readIban`** solo si tu integración necesita
  procesar IBANs (p. ej. para generar remesas SEPA fuera de
  FacturaDirecta). Por defecto, opera con `iban4` para minimizar el
  manejo de datos bancarios sensibles.

## Errores comunes

- `400 ValidationError` — `subtype: "manual"` con `contact` asignado.
- `400 ValidationError` — `subtype: "sepa*"` sin `contact`.
- `400 ValidationError` — `details` con estructura que no corresponde
  al `subtype` declarado.
- `400 ValidationError` — IBAN con formato inválido.
- `409 Conflict` — borrado de un método referenciado por documentos.

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
| GET | `/{companyId}/paymentMethods` | `getPaymentMethods` | `paymentMethods:read` | Lista de métodos de pago |
| GET | `/{companyId}/paymentMethods/{id}` | `getPaymentMethod` | `paymentMethods:read` | Obtener un método de pago |
| POST | `/{companyId}/paymentMethods` | `createPaymentMethod` | `paymentMethods:write` | Crear método de pago |
| PUT | `/{companyId}/paymentMethods/{id}` | `updatePaymentMethod` | `paymentMethods:write` | Actualizar método de pago |
| DELETE | `/{companyId}/paymentMethods/{id}` | `deletePaymentMethod` | `paymentMethods:write` | Borrar método de pago |

## Scopes

- **`paymentMethods:read`** — Lectura de métodos de pago.
- **`paymentMethods:write`** — Esctritura de métodos de pago.
