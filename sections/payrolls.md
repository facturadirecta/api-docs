---
title: Nóminas
audience:
  - developers
status: draft
---

# Nóminas

Una nómina (`payroll`) representa la **retribución mensual a un
empleado**: salario bruto, deducciones, retenciones de IRPF y
aportaciones de Seguridad Social, tanto del trabajador como de la
empresa. El recurso modela el documento contable de la nómina, no la
gestión laboral completa.

> En los ejemplos de esta página:
>
> - Los **UUIDs** (`par_…`, `con_…`, `ban_…`) son ilustrativos.
>   Cada empresa tiene los suyos; sustitúyelos por los identificadores
>   reales que devuelve la API.
> - El `contact` de una nómina es un contacto **con faceta `employee`**.
>   Si el contacto no tiene esa faceta configurada
>   (`accounts.employee` asignada), la creación falla.

## Flujos para crear nóminas

Puedes **construir el body manualmente** y enviarlo a
[`POST /payrolls`](#crear-nómina). Es el flujo descrito en esta
página.

## Estructura

- `content.type` — siempre `"payroll"`.
- `content.uuid` — identificador inmutable, prefijo `par_`.
- `content.main`:
  - `contact` — ID del empleado (contacto con faceta `employee`).
  - `date` — fecha de la nómina.
  - `dueDate` — fecha de vencimiento del pago.
  - `lines` — array de [líneas de nómina](#líneas-de-la-nómina)
    (mínimo 1).
  - `modelo190` — datos para el modelo 190 de retenciones IRPF (ver
    [modelo 190](#modelo-190)).
  - `paymentMethod` — método de pago.
  - `currency`, `total`, `totalBeforeTaxes`, etc.
- `content.attachments` — adjuntos (ver [Adjuntos](#adjuntos)).
- `content.accounting`, `content.books` — overrides contables
  específicos (uso avanzado).

### Líneas de la nómina

`content.main.lines` es un array con **al menos una línea**. Cada
línea representa un concepto de la nómina identificado por `type`.

Tipos admitidos (7):

| `type` | Significado | Cuenta contable típica |
|---|---|---|
| `debit` | Percepción salarial o no salarial. Con `debitType` se distingue. | 640* (salarios) |
| `credit` | Deducción (penalizaciones, anticipos, otras retenciones). | varía |
| `irpf_employee` | Retención de IRPF sobre percepciones dinerarias. | 475100 |
| `irpf_employee_kind` | Retención de IRPF sobre percepciones en especie. | 475100 |
| `irpf_employee_kind_passedon` | Retención IRPF percepciones en especie transferida al empleado. | 475100 |
| `ss_employee` | Aportación a la Seguridad Social a cargo del trabajador. | 476000 |
| `ss_company` | Aportación a la Seguridad Social a cargo de la empresa. | 642000 |

Campos de cada línea:

- `type` (obligatorio) — uno de los 7 anteriores.
- `account` (obligatorio) — cuenta contable.
- `lineTotal` (obligatorio) — importe.
- `base` (opcional) — base imponible, relevante para IRPF y SS.
- `debitType` — solo en `type: "debit"`, valores `"salary"` o
  `"nonSalary"`.
- `label` — etiqueta interna descriptiva.
- `text` — etiqueta libre.

**Las cuentas SS y el cargo a empresa**: `ss_company` aparece como
línea aunque no implica un cargo al empleado (es un gasto que asume la
empresa). El total de la nómina se calcula sobre lo que **percibe** el
empleado y las retenciones aplicadas; la línea `ss_company` queda
reflejada contablemente como gasto separado.

### modelo 190

`content.main.modelo190` describe la clave fiscal para la declaración
anual de retenciones (modelo 190 de Hacienda):

- `clavePercepcion` (obligatorio) — 1 letra mayúscula (A-Z) según el
  catálogo del modelo 190. La más común es **`A`** (rendimientos del
  trabajo en relación de dependencia).
- `subclavePercepcion` (opcional) — 2 dígitos.

Consulta el catálogo oficial del modelo 190 de la AEAT para la
correspondencia exacta entre claves y casos.

## Operaciones

- [Lista de nóminas](#lista-de-nóminas)
- [Crear nómina](#crear-nómina)
- [Obtener una nómina](#obtener-una-nómina)
- [Actualizar nómina](#actualizar-nómina)
- [Borrar nómina](#borrar-nómina)
- [Actualizar etiquetas](#actualizar-etiquetas)
- [Registrar pagos](#registrar-pagos)
- [Adjuntos](#adjuntos)

## Lista de nóminas

`GET /{companyId}/payrolls` devuelve las nóminas de la empresa,
paginadas.

**Parámetros de consulta específicos:**

- **`minDate`**, **`maxDate`** — rango por fecha de la nómina
  (`YYYY-MM-DD`).
- **`contact`** — ID del empleado.
- **`minTotal`**, **`maxTotal`** — rango de importe total.
- **`state`** — estado del pago. Es un array; admite combinar valores:
  - `pending` — no pagada y no vencida.
  - `overdue` — no pagada y vencida.
  - `paid` — pagada en su totalidad.
  - `overpaid` — pagada de más (importe registrado superior al total).
- **`allTheseTags`**, **`anyOfTheseTags`**, **`hasTags`** — filtros por etiquetas.
- **`sortBy`** — campo de orden.
- **`related`** — recursos a expandir.

**Parámetros globales aceptados:**

Acepta además los parámetros estándar `offset`, `limit`,
`minCreationDate`, `maxCreationDate`, `minModificationDate`,
`maxModificationDate` y el header `accept-version`. Ver
[Paginación](../guides/pagination.md) y
[Autenticación](../guides/authentication.md).

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/payrolls?state=pending&state=overdue&limit=50"
```

## Crear nómina

`POST /{companyId}/payrolls` crea una nómina. Soporta crear nómina y
sus pagos en una sola llamada (patrón "atómico").

**Parámetros del body:**

- **`content`** (obligatorio) — `PayrollWrite`. Estructura descrita en
  [Estructura](#estructura).
- **`tags`** (opcional).
- **`payments`** (opcional) — array de pagos a registrar en la misma
  llamada. Misma forma que el body de
  [`POST /payrolls/{id}/payments`](#registrar-pagos). Útil para
  crear nómina + pagos atómicamente y evitar dos llamadas. **No
  disponible en `POST /bills`** (donde los pagos requieren llamada
  separada).

**Notas:**

- `content.main.contact` debe apuntar a un contacto con faceta
  **`employee`** configurada (`accounts.employee` asignada). Si no,
  la API devuelve `400 ValidationError`.
- `lines` debe tener mínimo 1. Una nómina realista tiene típicamente
  4 líneas: `debit` (salario), `ss_employee`, `irpf_employee`,
  `ss_company` (ver ejemplo).
- La respuesta es la nómina creada completa, en el mismo formato que
  [Obtener una nómina](#obtener-una-nómina).

**Parámetros globales aceptados:** `accept-version`.

###### Ejemplo de request JSON

Nómina mínima heredada del ejemplo `minimal` del openapi (salario
bruto 2000 €, SS empleado 120, IRPF dinerario 240, SS empresa 600):

```json
{
  "content": {
    "type": "payroll",
    "main": {
      "date": "2026-05-31",
      "contact": "con_4f8e2a73-1d3c-4b5e-9c7a-8d6f2a1b3e5c",
      "modelo190": {
        "clavePercepcion": "A"
      },
      "lines": [
        {
          "type": "debit",
          "label": "Percepción salarial",
          "account": "640000",
          "lineTotal": 2000
        },
        {
          "type": "ss_employee",
          "label": "Aportación Seg.Soc. trabajador",
          "account": "476000",
          "base": 2000,
          "lineTotal": 120
        },
        {
          "type": "irpf_employee",
          "label": "Retenciones IRPF (Percepciones dinerarias)",
          "account": "475100",
          "base": 2000,
          "lineTotal": 240
        },
        {
          "type": "ss_company",
          "label": "Aportaciones Cotización Seg.Soc. Empresa",
          "account": "642000",
          "base": 2000,
          "lineTotal": 600
        }
      ]
    }
  }
}
```

Crear nómina + pago en la misma llamada (atómico):

```json
{
  "content": { "...": "como arriba" },
  "payments": [
    { "bank": "ban_5e7d8a31-9c4b-4f6e-a1d3-2b5c7e9f1a4d", "date": "2026-06-05" }
  ]
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '@payroll.json' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/payrolls"
```

## Obtener una nómina

`GET /{companyId}/payrolls/{id}` devuelve una nómina por ID.

**Parámetros:**

- `related` (opcional) — recursos relacionados a expandir, por ejemplo
  `payments` para ver los pagos registrados.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/payrolls/par_8c3f1e29-4a7b-4d6e-9c2a-5f8d1b3e7a4c?related=payments"
```

## Actualizar nómina

`PUT /{companyId}/payrolls/{id}` sustituye **el contenido completo**
de la nómina. No es un PATCH.

**Restricciones:**

- Si la nómina tiene pagos asociados, las modificaciones pueden estar
  limitadas. Antes de actualizar, consulta sus pagos en `related`.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '@payroll.json' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/payrolls/par_8c3f1e29-4a7b-4d6e-9c2a-5f8d1b3e7a4c"
```

## Borrar nómina

`DELETE /{companyId}/payrolls/{id}` elimina una nómina.

**Restricciones:**

- Si la nómina tiene pagos asociados, la API rechaza el borrado con
  `409 Conflict`. Elimina primero los pagos.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/payrolls/par_8c3f1e29-4a7b-4d6e-9c2a-5f8d1b3e7a4c"
```

## Actualizar etiquetas

`PUT /{companyId}/payrolls/{id}/tags` reemplaza el conjunto de etiquetas
sin tocar el resto del contenido.

###### Ejemplo de request JSON

```json
{
  "tags": ["nominas-2026", "departamento-tecnico"]
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '{"tags":["nominas-2026","departamento-tecnico"]}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/payrolls/par_8c3f1e29-4a7b-4d6e-9c2a-5f8d1b3e7a4c/tags"
```

## Registrar pagos

`POST /{companyId}/payrolls/{id}/payments` registra uno o varios pagos
contra una nómina. El body es un **array** de pagos, equivalente a
[Registrar pagos en bills](./bills.md#registrar-pagos).

**Parámetros de cada pago:**

- `bank` (obligatorio) — ID del banco/cuenta de tesorería donde se
  anota el pago.
- `amount` (opcional) — importe en la moneda de la nómina. Si se
  omite, salda **todo el pendiente**. Solo un pago del array puede
  omitir `amount`, y debe ser el **último**.
- `date` (opcional) — fecha del pago. Por defecto la fecha de la
  nómina cuando se crea, la fecha actual cuando se actualiza.
- `companyCurrencyAmount` (opcional) — importe contable cuando la
  moneda de la nómina difiere de la contable.
- `description`, `notes` (opcionales).

**Respuesta:** la nómina completa actualizada con los pagos registrados
en `related.payments`.

**Para crear nómina y pago a la vez**, usa el campo `payments` en el
body de [`POST /payrolls`](#crear-nómina) — único patrón del recurso
no disponible en bills.

###### Ejemplo de request JSON

Pago único que salda toda la nómina:

```json
[
  { "bank": "ban_5e7d8a31-9c4b-4f6e-a1d3-2b5c7e9f1a4d" }
]
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '[{"bank":"ban_5e7d8a31-9c4b-4f6e-a1d3-2b5c7e9f1a4d"}]' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/payrolls/par_8c3f1e29-4a7b-4d6e-9c2a-5f8d1b3e7a4c/payments"
```

## Adjuntos

Una nómina puede tener adjuntos vinculados (típicamente el PDF
original de la nómina). Los adjuntos **no se suben en el body**:
primero se crean con `POST /uploads` y después se vinculan con
`POST /payrolls/{id}/attachments`.

### Listar adjuntos

`GET /{companyId}/payrolls/{id}/attachments` devuelve los adjuntos
vinculados.

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/payrolls/par_8c3f1e29-4a7b-4d6e-9c2a-5f8d1b3e7a4c/attachments"
```

### Vincular adjuntos

`POST /{companyId}/payrolls/{id}/attachments` añade adjuntos a partir
de uploads previos.

**Parámetros del body:**

- `uploadIds` — array de `upl_<uuid>`. Mínimo uno.

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"uploadIds":["upl_6f8e9b42-1d3c-4a7f-b2e4-9c1d5e7f3a6b"]}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/payrolls/par_8c3f1e29-4a7b-4d6e-9c2a-5f8d1b3e7a4c/attachments"
```

### Eliminar un adjunto

`DELETE /{companyId}/payrolls/{id}/attachments/{attachmentIndex}`
elimina un adjunto. `attachmentIndex` es la posición en el array
(basada en cero).

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/payrolls/par_8c3f1e29-4a7b-4d6e-9c2a-5f8d1b3e7a4c/attachments/0"
```

## Recomendaciones

- **Crear nómina + pago en una sola llamada**: usa el campo
  `payments` en el body de `POST /payrolls` cuando ya conoces el
  pago. Evita una segunda petición y deja la nómina marcada como
  `paid` desde el inicio.
- **`modelo190.clavePercepcion: "A"`** cubre el caso más habitual
  (relación laboral común). Para autónomos, becarios, miembros de
  consejos de administración, etc., consulta el catálogo del
  modelo 190 de la AEAT.
- **No olvides la línea `ss_company`**: aunque no sale del salario
  del empleado, sí es un gasto contable de la empresa y debe quedar
  reflejado en la nómina.

## Errores comunes

- `400 ValidationError` — `contact` sin faceta `employee` configurada.
- `400 ValidationError` — `lines` vacío (debe tener al menos 1).
- `400 ValidationError` — `modelo190.clavePercepcion` con valor
  fuera del catálogo (A-Z).
- `409 Conflict` — borrado de una nómina con pagos asociados.
- `409 Conflict` — actualización incompatible con pagos existentes.

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
| GET | `/{companyId}/payrolls` | `getPayrolls` | `payrolls:read` | Lista de nóminas |
| GET | `/{companyId}/payrolls/{id}` | `getPayroll` | `payrolls:read` | Obtener una nómina |
| GET | `/{companyId}/payrolls/{id}/attachments` | `getPayrollAttachments` | `payrolls:read` | Listar adjuntos de una nómina |
| POST | `/{companyId}/payrolls` | `createPayroll` | `payrolls:write` | Crear nómina |
| POST | `/{companyId}/payrolls/{id}/attachments` | `addPayrollAttachments` | `payrolls:write` | Vincular adjuntos a una nómina |
| POST | `/{companyId}/payrolls/{id}/payments` | `createPayrollPayments` | `payrolls:write` | Crear pagos para una nómina |
| PUT | `/{companyId}/payrolls/{id}` | `updatePayroll` | `payrolls:write` | Actualizar nómina |
| PUT | `/{companyId}/payrolls/{id}/tags` | `updatePayrollTags` | `payrolls:write` | Actualizar etiquetas de nómina |
| DELETE | `/{companyId}/payrolls/{id}` | `deletePayroll` | `payrolls:write` | Borrar nómina |
| DELETE | `/{companyId}/payrolls/{id}/attachments/{attachmentIndex}` | `deletePayrollAttachment` | `payrolls:write` | Eliminar adjunto de una nómina |

## Scopes

- **`payrolls:read`** — Lectura de nóminas.
- **`payrolls:write`** — Modificación de nóminas.
