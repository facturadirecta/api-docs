---
title: Extractos bancarios
audience:
  - developers
status: draft
---

# Extractos bancarios

Un **extracto bancario** (`statement`) es un movimiento individual de
una cuenta bancaria sincronizada con FacturaDirecta. Cada cargo o
ingreso del banco genera un statement con su fecha, importe y concepto.

El recurso se centra en la **conciliación**: vincular cada statement
con el asiento contable correspondiente (cobro de factura, pago de
gasto, etc.) para mantener la contabilidad alineada con el banco.

> En los ejemplos de esta página, los UUIDs (`sta_…`, `tra_…`,
> `inv_…`, `bil_…`, `ban_…`) son ilustrativos. Cada empresa tiene los
> suyos; sustitúyelos por los identificadores reales que devuelve la
> API.

## Los statements no se crean por API

Los statements los **importa el sistema automáticamente** desde la
integración bancaria configurada en la empresa (Afterbanks, ArcoPay,
importación de extractos en CSV, etc.). La API pública solo expone
**lectura y conciliación**: no hay `POST /statements` ni
`DELETE /statements/{id}`. Para crear un statement nuevo hay que pasar
por los flujos de importación bancaria de la interfaz.

## Estados del statement

`state` toma tres valores:

- **`pending`** — recién importado, sin conciliar. Aparece en el panel
  de conciliación pendiente.
- **`reconciled`** — vinculado a un asiento contable. Su importe ya
  está reflejado en la contabilidad.
- **`ignored`** — marcado para no aparecer en pendientes (transferencias
  entre cuentas propias, comisiones que no se contabilizan, etc.).

El campo `ignore` complementa: cuando un statement está marcado como
ignorado, `state` pasa a `ignored` y el statement queda fuera de los
listados por defecto.

## Estructura

`StatementContent`:

| Campo | Tipo | Significado |
|---|---|---|
| `uuid` | string | Identificador del statement, prefijo `sta_`. |
| `date` | date | Fecha de la operación bancaria. |
| `title` | string | Concepto del movimiento (lo que dice el banco). |
| `amount` | number | Importe. **Positivo para ingresos, negativo para cargos**. |
| `currency` | string | Moneda del movimiento (ISO 4217). |
| `companyCurrencyAmount` | number | Importe convertido a la moneda de la empresa. |
| `state` | enum | `pending`/`reconciled`/`ignored`. |
| `bank` | string | ID del banco propio del statement (`ban_*`). |
| `ignore` | boolean | Si está marcado como ignorado. |
| `matchedDocument` | string \| null | ID del asiento contable (`tra_*`) con el que está conciliado el movimiento. Es `null` cuando no está conciliado o cuando el asiento se ha archivado. |
| `notes` | string | Notas libres. |
| `sourceKey` | string | Identificador del statement en la fuente externa (banco). |

## Operaciones

- [Lista de extractos](#lista-de-extractos)
- [Obtener un extracto](#obtener-un-extracto)
- [Conciliar con asiento existente](#conciliar-con-asiento-existente)
- [Conciliar creando nuevo asiento](#conciliar-creando-nuevo-asiento)
- [Desconciliar](#desconciliar)

## Lista de extractos

`GET /{companyId}/statements` devuelve los extractos bancarios de la
empresa, paginados.

**Parámetros de consulta específicos:**

- **`minDate`**, **`maxDate`** — rango por fecha de la operación.
- **`bank`** — ID del banco propio (`ban_*`).
- **`currency`** — código ISO 4217.
- **`minAmount`**, **`maxAmount`** — rango de importe.
- **`matchedDocument`** — ID de un asiento contable (`tra_*`). Devuelve
  el movimiento conciliado con ese asiento. Si el asiento está archivado o
  no existe, el resultado queda vacío.
- **`state`** — filtrar por estado (`pending`, `reconciled`, `ignored`).
- **`sortBy`** — campo de orden.

**Parámetros globales aceptados:**

Acepta además los parámetros estándar `offset`, `limit`,
`minCreationDate`, `maxCreationDate`, `minModificationDate`,
`maxModificationDate` y el header `accept-version`. Ver
[Paginación](../guides/pagination.md) y
[Autenticación](../guides/authentication.md).

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/statements?state=pending&bank=ban_5e7d8a31-9c4b-4f6e-a1d3-2b5c7e9f1a4d&limit=50"
```

Movimiento conciliado con un asiento concreto:

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/statements?matchedDocument=tra_3a8f1e29-4d7b-4c5e-9a2f-6d8b1c4e7a5f"
```

## Obtener un extracto

`GET /{companyId}/statements/{id}` devuelve un statement por ID.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/statements/sta_2c7f4e91-8d3a-4b5c-9e1f-6a2d8b4c3e5f"
```

## Conciliar con asiento existente

`PUT /{companyId}/statements/{id}/reconciliation` vincula el statement
con un asiento contable (`transaction`) **que ya existe** en
contabilidad. Útil cuando el cobro o pago ya está registrado en la BD
(por ejemplo, porque registraste el pago de una factura con
`POST /bills/{id}/payments` y luego importas el extracto bancario).

**Parámetros del body:**

- **`transactionId`** (obligatorio) — ID del asiento contable a
  vincular (`tra_*`).

**Notas:**

- Tras la conciliación, `state` pasa a `reconciled`.
- El asiento debe tener un importe compatible con el del statement.

**Parámetros globales aceptados:** `accept-version`.

###### Ejemplo de request JSON

```json
{
  "transactionId": "tra_3a8f1e29-4d7b-4c5e-9a2f-6d8b1c4e7a5f"
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '{"transactionId":"tra_3a8f1e29-4d7b-4c5e-9a2f-6d8b1c4e7a5f"}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/statements/sta_2c7f4e91-8d3a-4b5c-9e1f-6a2d8b4c3e5f/reconciliation"
```

## Conciliar creando nuevo asiento

`POST /{companyId}/statements/{id}/reconciliation` concilia el
statement **creando un asiento contable nuevo** definido por las líneas
que envías. Útil cuando el cobro o pago aún no está registrado y
quieres reflejarlo en la contabilidad como parte de la conciliación.

**Parámetros del body:**

- **`lines`** (obligatorio) — array de líneas del asiento contable.
  Cada línea tiene:
  - `account` (obligatorio) — cuenta contable.
  - `document` (opcional) — ID del documento al que se imputa (factura,
    gasto, banco propio).
  - `debit` o `credit` (uno de los dos) — importe en la moneda de la
    contabilidad de la empresa.
  - `currencyDebit` o `currencyCredit` — importe en la moneda original
    del movimiento (puede coincidir con `debit`/`credit` si la moneda
    es la misma).
  - `currency` — código ISO 4217.

Las líneas deben **cuadrar contablemente**: total de debits = total
de credits.

**Notas:**

- Tras la conciliación, `state` pasa a `reconciled`.
- El asiento creado se vincula automáticamente al statement.

**Parámetros globales aceptados:** `accept-version`.

###### Ejemplo de request JSON

Conciliar un ingreso (cobro de factura de venta), heredado del ejemplo
`reconcileReceive` del openapi:

```json
{
  "lines": [
    {
      "account": "572000",
      "document": "ban_5e7d8a31-9c4b-4f6e-a1d3-2b5c7e9f1a4d",
      "debit": 121,
      "credit": null,
      "currencyDebit": 121,
      "currencyCredit": null,
      "currency": "EUR"
    },
    {
      "account": "430000",
      "document": "inv_8a3c2f15-7b4e-4d6a-9c1f-5e2d8b3c4a7f",
      "debit": null,
      "credit": 121,
      "currencyDebit": null,
      "currencyCredit": 121,
      "currency": "EUR"
    }
  ]
}
```

Conciliar un cargo (pago de factura de compra), heredado del ejemplo
`reconcileSend`:

```json
{
  "lines": [
    {
      "account": "572000",
      "document": "ban_5e7d8a31-9c4b-4f6e-a1d3-2b5c7e9f1a4d",
      "debit": null,
      "credit": 121,
      "currencyDebit": null,
      "currencyCredit": 121,
      "currency": "EUR"
    },
    {
      "account": "400000",
      "document": "bil_4a9c2e58-3b1f-4d6a-8e2c-7f1a9b3d5c4e",
      "debit": 121,
      "credit": null,
      "currencyDebit": 121,
      "currencyCredit": null,
      "currency": "EUR"
    }
  ]
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '@reconcile.json' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/statements/sta_2c7f4e91-8d3a-4b5c-9e1f-6a2d8b4c3e5f/reconciliation"
```

## Desconciliar

`DELETE /{companyId}/statements/{id}/reconciliation` rompe el vínculo
entre el statement y su asiento contable. El statement vuelve a estado
`pending`.

**Notas:**

- El asiento contable asociado **no se borra**, solo se desvincula del
  statement. Si el asiento fue creado por
  `reconcileStatementWithNewTransaction`, puede quedar huérfano; bórralo
  manualmente si ya no aplica.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/statements/sta_2c7f4e91-8d3a-4b5c-9e1f-6a2d8b4c3e5f/reconciliation"
```

## Recomendaciones

- **Para conciliar pagos ya registrados** (típico: pago de factura ya
  hecho con `POST /bills/{id}/payments`), usa
  [conciliar con asiento existente](#conciliar-con-asiento-existente).
  El asiento ya existe; solo hay que enlazarlo.
- **Para conciliar movimientos sin asiento previo** (transferencias
  varias, comisiones, devoluciones), usa
  [conciliar creando nuevo asiento](#conciliar-creando-nuevo-asiento)
  enviando las líneas contables del asiento a crear.
- **Para localizar el movimiento de un asiento conciliado**, usa
  `matchedDocument=<id-asiento>`. El filtro solo devuelve una relación
  vigente. Si archivas el asiento, el movimiento vuelve a estar pendiente,
  `matchedDocument` pasa a `null` en la respuesta y el filtro deja de
  devolverlo.
- **No abusar de `ignored`**: úsalo para movimientos que realmente no
  deben afectar a contabilidad (transferencias entre cuentas propias,
  movimientos de cobertura). Para movimientos relevantes, concilia
  aunque sea con un asiento contable simple.

## Errores comunes

- `400 ValidationError` — líneas que no cuadran (debits ≠ credits) en
  `reconcileWithNewTransaction`.
- `400 ValidationError` — `transactionId` apunta a un asiento con
  importe incompatible con el statement.
- `404 Not Found` — `transactionId` no existe en la empresa.
- `409 Conflict` — intento de conciliar un statement que ya está
  conciliado. Desconcilia primero.

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
| GET | `/{companyId}/statements` | `getStatements` | `statements:read` | Lista de extractos bancarios |
| GET | `/{companyId}/statements/{id}` | `getStatement` | `statements:read` | Obtener un extracto bancario |
| POST | `/{companyId}/statements/{id}/reconciliation` | `reconcileStatementWithNewTransaction` | `statements:write` | Conciliar extracto bancario creando un nuevo asiento contable |
| PUT | `/{companyId}/statements/{id}/reconciliation` | `reconcileStatement` | `statements:write` | Conciliar extracto bancario con asiento contable existente |
| DELETE | `/{companyId}/statements/{id}/reconciliation` | `unreconcileStatement` | `statements:write` | Desconciliar extracto bancario |

## Scopes

- **`statements:read`** — Lectura de extractos bancarios.
- **`statements:write`** — Conciliación de extractos bancarios.
