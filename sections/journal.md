---
title: Diario contable
audience:
  - developers
status: draft
---

# Diario contable

El **diario contable** es la lista cronológica de apuntes (líneas
de asiento) generados por todos los documentos contabilizables de la
empresa: facturas de venta, facturas de compra, nóminas, asientos
manuales, pagos/cobros, modelos de impuestos, amortizaciones, etc.

Cada item del diario es un **apunte individual**: una línea de un
asiento con su cuenta, debe/haber, fecha y referencia al documento
que la origina. La API expone una sola operación de **lectura
paginada**: la contabilidad se construye desde los documentos
(facturas, nóminas, etc.), no creando apuntes sueltos. Para alterar
el diario, hay que crear o modificar el documento de origen.

## Estructura de un apunte

| Campo | Tipo | Significado |
|---|---|---|
| `id` | string | ID del documento que origina el apunte (puede ser `inv_*`, `bil_*`, `pay_*`, `tra_*`, etc.). |
| `idType` | enum | Tipo del documento origen. Valores: `bill`, `invoice`, `payroll`, `payment`, `taxes`, `transaction`. |
| `idTitle` | string | Título del documento origen. |
| `idSummary` | string | Resumen/descripción del documento origen (disponible para facturas de venta, compras y nóminas). |
| `idLineIndex` | integer | Número de orden del apunte dentro del asiento del documento. |
| `idAccounting` | object | Asiento contable completo del documento origen. **Solo si se pide `related=idAccounting`.** |
| `date` | date | Fecha del apunte. |
| `debit` | number | Importe en el DEBE en EUR. |
| `credit` | number | Importe en el HABER en EUR. |
| `currency` | string | Moneda original del apunte (ISO 4217). **Solo si la empresa tiene multimoneda activo y se pide `related=currency`.** |
| `currencyDebit` | number | DEBE en moneda original (mismas condiciones que `currency`). |
| `currencyCredit` | number | HABER en moneda original (mismas condiciones que `currency`). |
| `account` | string | Cuenta contable (6 dígitos del plan contable de la empresa). |
| `accountTitle` | string | Nombre de la cuenta en el plan. |
| `document` | string | Documento que actúa como subcuenta (si aplica). |
| `documentType` | enum | Tipo del documento subcuenta. Valores: `bill`, `invoice`, `payroll`, `payment`, `transaction`, `bank`, `contact`, `product`. |
| `documentTitle` | string | Título del documento subcuenta. |
| `journalPriority` | integer | Orden del asiento dentro del día. Ver tabla más abajo. |
| `tags` | string[] | Etiquetas del documento origen. **Solo si se pide `related=tags`.** |

> Los campos `debit`/`credit`/`currency*` siguen la convención
> contable española: positivos en la columna que corresponda; nunca
> negativos para "contraapuntes". Para neutralizar un asiento se
> genera otro con los lados invertidos.

## `journalPriority`: orden de asientos dentro del día

Los asientos del diario tienen una prioridad numérica entre 1 y 999
que determina su orden dentro de una misma fecha. Los valores que
emite FacturaDirecta son:

| Prioridad | Tipo de asiento |
|---|---|
| **1** | Inicio de contabilidad / Apertura de periodo |
| **2** | Asiento ficticio generado por un documento con fecha previa al inicio de contabilidad |
| **3** | Asiento de traspaso de beneficios en la apertura |
| **100** | Asiento estándar (la mayoría de movimientos) |
| **970** | Amortizaciones |
| **980** | Regularización de moneda extranjera |
| **990** | Cierre de grupos 6 y 7 |
| **999** | Cierre del periodo |

El filtro `journalPriority` permite aislar uno de estos tipos (por
ejemplo, `journalPriority=970` para obtener solo amortizaciones).

## Operaciones

- [Lista de apuntes del diario](#lista-de-apuntes-del-diario)

## Lista de apuntes del diario

`GET /{companyId}/journal` devuelve la lista de apuntes del diario
paginada por la línea individual (no por asiento).

**Parámetros de consulta específicos:**

- **`minDate`**, **`maxDate`** — rango por fecha del apunte
  (`YYYY-MM-DD`, inclusivos).
- **`account`** — cuenta contable. Acepta entre 1 y 6 dígitos:
  - entre 1 y 5 dígitos: prefijo (`6` → todos los gastos; `64` →
    todas las cuentas de personal).
  - 6 dígitos: cuenta completa (`700000` → solo apuntes de la cuenta
    de ventas 700000).
- **`id`** — ID del documento que origina los apuntes. Devuelve todas
  las líneas de ese asiento, factura, gasto, nómina o remesa.
- **`document`** — ID del documento que actúa como subcuenta. Filtra
  los apuntes que afectan al saldo de ese contacto, banco o producto.
  No equivale a `id`: un mismo asiento puede tener líneas imputadas a
  documentos distintos.
- **`journalPriority`** — entero. Aísla un tipo concreto de asiento
  (ver tabla arriba).
- **`sortBy`** — solo se acepta `date` o `-date`. El orden secundario
  es siempre por `journalPriority`, `id` y `line_id` (ambos en la
  misma dirección).
- **`related`** — array. Permite añadir datos derivados al item:
  - `idAccounting` — incluye el asiento contable completo del
    documento origen en cada apunte. Útil cuando vas a procesar
    asientos enteros.
  - `tags` — incluye el array `tags` (etiquetas del documento
    origen).
  - `currency` — incluye los campos `currency`, `currencyDebit` y
    `currencyCredit` con el contravalor en moneda original.
    **Solo aplica si la empresa tiene multimoneda activo**; en caso
    contrario, los campos se siguen omitiendo aunque pidas `related`.

**Filtros por etiquetas:**

Soporta los parámetros estándar `allTheseTags`, `anyOfTheseTags` y
`hasTags` para filtrar por etiquetas del documento que origina el
apunte. Ver [Etiquetas en otros documentos] para la semántica.

**Parámetros globales aceptados:**

Acepta además los parámetros estándar `offset`, `limit` y el header
`accept-version`. Ver [Paginación](../guides/pagination.md) y
[Autenticación](../guides/authentication.md).

> Este endpoint **no** acepta los filtros estándar
> `minCreationDate`/`minModificationDate` (no son apuntes con vida
> propia: dependen del documento de origen).

###### Copy as cURL

Saldo de la cuenta 572000 en un rango de fechas:

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/journal?account=572000&minDate=2025-01-01&maxDate=2025-12-31&sortBy=date"
```

Solo amortizaciones de un año:

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/journal?journalPriority=970&minDate=2025-01-01&maxDate=2025-12-31"
```

Todos los apuntes que origina un asiento contable:

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/journal?id=tra_3a8f1e29-4d7b-4c5e-9a2f-6d8b1c4e7a5f"
```

Apuntes que afectan a un contacto, con asiento completo y etiquetas:

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/journal?document=con_2e3f4a18-9b7c-4d6a-8e1f-5c2d3b4a7e9f&related=idAccounting&related=tags"
```

## Recomendaciones

- **Lectura por rangos de fecha**: el diario crece sin límite. Pagina
  siempre con `minDate`/`maxDate` razonables, no pidas el diario
  entero. Para exportaciones, itera mes a mes o trimestre a
  trimestre.
- **Para saldos de cuenta**: usa `account=<6 dígitos>` y suma
  `debit - credit` (o `credit - debit` según el tipo de cuenta) en
  el cliente. La API no agrega saldos.
- **Para recuperar un asiento completo**: filtra por `id`. No filtres
  por `document`, porque ese parámetro selecciona las líneas imputadas a
  una subcuenta y puede dejar fuera otros apuntes del mismo asiento.
- **Para vincular con documentos**: el `id` del apunte es el ID del
  documento origen. Puedes pedir su detalle con la operación de
  lectura del recurso correspondiente
  ([invoices](./invoices.md), [bills](./bills.md),
  [payrolls](./payrolls.md), etc.).
- **`related=idAccounting` es caro**: solo pídelo si vas a procesar
  asientos completos. Si solo necesitas apuntes individuales, no lo
  uses.
- **Multimoneda**: si tu empresa no tiene multimoneda activado, no
  uses `related=currency` (no devuelve nada). Si lo tiene, los
  campos `currencyDebit`/`currencyCredit` son útiles para informes
  en la moneda original del movimiento.

## Errores comunes

- `400 ValidationError` (`code: "invalidParams"`) — valor incorrecto
  para `sortBy`. Solo se acepta `date` o `-date`.
- `400 ValidationError` — `account` con formato distinto a 1-6
  dígitos.
- `403 Forbidden` — falta el scope `accounting:read`.

Ver [Errores y validaciones](../guides/errors.md) para el formato
general.

## Referencia exhaustiva

Esta página cubre los matices funcionales y los casos típicos. Para
el contrato completo del item (incluyendo el schema `Accounting`
devuelto por `related=idAccounting`), consulta el
[Swagger UI](https://www.facturadirecta.com/api) o el
[openapi crudo](https://app.facturadirecta.com/openapi.json).

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| GET | `/{companyId}/journal` | `getJournal` | `accounting:read` | Diario contable |

## Scopes

- **`accounting:read`** — Lectura de datos de contabilidad (diario).
