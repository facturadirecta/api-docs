---
title: Informes
audience:
  - developers
status: draft
---

# Informes

Los **informes** devuelven, ya calculadas, las mismas cifras que los
informes de la web. Son de solo lectura y se generan en el momento a
partir de la contabilidad de la empresa.

El **resumen de resultados** responde a «cuánto he facturado, cuánto he
gastado y cuánto he ganado» en un periodo, separando cada importe por su
origen: facturas, gastos, nóminas, amortizaciones, diferencias de cambio
y otros. Está disponible en **todos los planes** y da las mismas cifras
que las tarjetas de Ingresos, Gastos y Beneficio del panel.

Para ver qué documentos componen una cifra, el **detalle del resumen**
lista los documentos de una categoría con el importe que aporta cada uno.

Los **informes contables** (pérdidas y ganancias, balance de situación y
sumas y saldos) siguen el modelo oficial del Plan General Contable y
necesitan el **módulo de Contabilidad**, como en la web.

> Para totales usa estos informes y no sumes apuntes del
> [diario](./journal.md): los informes aplican las mismas reglas que la
> web (por ejemplo, excluyen el asiento de regularización), y una suma
> propia puede no coincidir con lo que ve el usuario.

## Estructura del resumen de resultados

La respuesta tiene `meta` y `rows`:

| Campo | Tipo | Significado |
|---|---|---|
| `meta.report` | string | Siempre `resultsSummary`. |
| `meta.currency` | string | Moneda de la empresa (ISO 4217). Todos los importes van en ella. |
| `meta.columns` | object[] | Columnas del informe, ver más abajo. |
| `meta.generatedAt` | date-time | Momento en que se calculó el informe. |
| `rows` | object[] | Filas del informe, en orden de presentación. |

Cada fila de `rows`:

| Campo | Tipo | Significado |
|---|---|---|
| `id` | enum | Categoría (`invoices`, `currencyGains`, `otherIncome`, `bills`, `payrolls`, `depreciation`, `currencyLosses`, `otherExpenses`) o total (`income`, `expenses`, `profit`). |
| `kind` | enum | `category` o `total`. |
| `side` | enum | Solo en las categorías: `income` (ingresos) o `expenses` (gastos). |
| `title` | string | Nombre de la fila en el idioma de la petición. |
| `values` | object | Valor de cada columna, por su `id`. |

Las filas aparecen en este orden: categorías de ingresos, total de
ingresos, categorías de gastos, total de gastos y beneficio. Una categoría
solo aparece si tiene importe en alguna columna; los tres totales
aparecen siempre.

Los importes son positivos cuando suman a su lado: un ingreso de 100 €
vale `100` en ingresos y un gasto de 40 € vale `40` en gastos. El
beneficio es el total de ingresos menos el total de gastos.

### Columnas

Cada columna de `meta.columns` tiene un `id`, que es la clave de `values`
en cada fila, y un `kind`:

| `kind` | Campos | Valor en cada fila |
|---|---|---|
| `movement` | `start`, `end` | Importe del periodo, con las dos fechas incluidas. |
| `difference` | `current`, `reference` | Importe de `current` menos el de `reference`. |
| `varianceRatio` | `current`, `reference` | Variación como fracción: `0.2` es un 20 %. Vale `null` si la referencia es cero. |

Sin opciones de comparativa hay una sola columna, `current`. Con `compare`
las columnas son `current` y `reference`, más `difference` y `variance` si
se piden. Con `breakdown` son `p0`, `p1`... por orden de fechas, más
`total` si se pide.

### Qué importes entran

El resumen suma los apuntes de las cuentas de los grupos 6 (gastos) y 7
(ingresos) del periodo, sin el asiento de regularización de fin de año.
Por eso coincide con sumas y saldos de esos grupos y con el resultado de
pérdidas y ganancias excluyendo la regularización.

Cada importe va a una categoría:

1. Las cuentas de diferencias de cambio de la configuración contable
   (768000 y 668000 por defecto) van a `currencyGains` y
   `currencyLosses`, sea cual sea el documento que las genera.
2. El resto se reparte por el documento que genera el asiento: las
   facturas de venta en `invoices`, los gastos en `bills` y las nóminas
   en `payrolls`.
3. Lo que queda del grupo 68 va a `depreciation`.
4. Todo lo demás va a `otherIncome` u `otherExpenses`.

## Estructura de los informes contables

Pérdidas y ganancias y el balance de situación tienen `meta`, `rows` y
`diagnostics`. `meta` añade a lo del resumen:

| Campo | Tipo | Significado |
|---|---|---|
| `meta.template` | object | Modelo oficial: `id`, `variant` (`pyme`, o `pymesfl` para entidades sin fines lucrativos, según la empresa) y `version`. |
| `meta.detail` | enum | Nivel de detalle aplicado: `summary`, `accounts` o `full`. |
| `meta.policies` | object | `includeClosure`, `excludeRevenueClearing` (solo pérdidas y ganancias) y `tags`, si se filtró por etiquetas. |

Cada fila de `rows`:

| Campo | Tipo | Significado |
|---|---|---|
| `id` | string | Identificador estable: el epígrafe del modelo (`A_1`, `PC_2_3`…). Un grupo añade su prefijo de cuenta (`A_1/700`) y una fila de detalle, su cuenta y tercero (`A_1/700/700000/con_…`). |
| `parentId` | string | Fila cuyo total incluye esta. Puede faltar en `rows` si todas sus columnas son cero. |
| `kind` | enum | `accounts` o `sum` (epígrafes), `group` (grupo de cuentas), `detail` (cuenta y tercero), `unregularizedResult` (resultado aún sin regularizar, en el balance) o `imbalance` (descuadre del balance). |
| `level` | integer | Nivel de sangría, el de la web. |
| `title` | string | Título en el idioma de la petición. |
| `accountPrefix` | string | Prefijo de cuenta de un grupo. |
| `account`, `accountTitle` | string | Cuenta de una fila de detalle y su título en el plan contable. |
| `thirdParty` | object | Tercero de una fila de detalle: `id`, `type` y `name`. |
| `values` | object | Valor de cada columna por su `id`. |

Una fila cuyas columnas son todas cero no aparece, igual que en la web.
En el balance, cada columna es de tipo `balanceAt`: el saldo acumulado a
su `date`. Con `revenueShare`, pérdidas y ganancias añade columnas
`shareOfRow` con el peso de cada fila sobre el importe neto de la cifra
de negocios.

Cada aviso de `diagnostics` tiene `code`, `severity` (`info` o
`warning`), `message` en el idioma de la petición y, según el caso,
`years` o `accounts`. Son los mismos avisos que muestra la web, por
ejemplo un ejercicio anterior sin regularizar (`previousYearNotClosed`)
o cuentas con saldo fuera del modelo (`unmappedBalances`).

Sumas y saldos tiene `meta`, `rows`, `totals` y `pagination`. Cada fila
es un subtotal (`group`, con `accountPrefix`), una cuenta (`account`) o
un tercero de una cuenta (`thirdParty`), con sus importes en `amounts`:
`previousBalance`, `debit`, `credit` y `balance`. `totals` es el total
del informe completo, no solo de la página.

## Operaciones

- [Obtener el resumen de resultados](#obtener-el-resumen-de-resultados)
- [Listar los documentos de una categoría](#listar-los-documentos-de-una-categoría)
- [Obtener pérdidas y ganancias](#obtener-pérdidas-y-ganancias)
- [Obtener el balance de situación](#obtener-el-balance-de-situación)
- [Obtener sumas y saldos](#obtener-sumas-y-saldos)

## Obtener el resumen de resultados

`GET /{companyId}/reports/resultsSummary`

| Parámetro | Valores | Por defecto | Significado |
|---|---|---|---|
| `startDate`, `endDate` | fecha `YYYY-MM-DD` | año natural en curso | Periodo del informe, con las dos fechas incluidas. Como máximo, 5 años. |
| `compare` | `none`, `previousPeriod`, `previousYear` | `none` | Añade un periodo de referencia: el anterior de la misma duración o el mismo periodo del año anterior. |
| `breakdown` | `none`, `month`, `quarter`, `year` | `none` | Divide el periodo en meses, trimestres o años naturales, como máximo 24 columnas. |
| `difference` | booleano | `false` | Con `compare`, añade la columna `difference`. |
| `variance` | booleano | `false` | Con `compare`, añade la columna `variance`. |
| `total` | booleano | `false` | Con `breakdown`, añade la columna `total` con el periodo completo. |

`compare` y `breakdown` no se combinan. `difference` y `variance`
necesitan `compare`, y `total` necesita `breakdown`. Cualquier combinación
no válida responde `400`.

### Ejemplo: ingresos, gastos y beneficio del año comparados con el anterior

```bash
curl "https://app.facturadirecta.com/api/$COMPANY_ID/reports/resultsSummary?startDate=2025-01-01&endDate=2025-12-31&compare=previousYear&difference=true" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

```json
{
  "meta": {
    "report": "resultsSummary",
    "currency": "EUR",
    "columns": [
      { "id": "current", "kind": "movement", "start": "2025-01-01", "end": "2025-12-31" },
      { "id": "reference", "kind": "movement", "start": "2024-01-01", "end": "2024-12-31" },
      { "id": "difference", "kind": "difference", "current": "current", "reference": "reference" }
    ],
    "generatedAt": "2025-12-31T10:00:00.000Z"
  },
  "rows": [
    { "id": "invoices", "kind": "category", "side": "income", "title": "Ventas facturadas", "values": { "current": 98000, "reference": 91500, "difference": 6500 } },
    { "id": "currencyGains", "kind": "category", "side": "income", "title": "Diferencias de cambio a favor", "values": { "current": 1200, "reference": 0, "difference": 1200 } },
    { "id": "income", "kind": "total", "title": "Total ingresos", "values": { "current": 99200, "reference": 91500, "difference": 7700 } },
    { "id": "bills", "kind": "category", "side": "expenses", "title": "Compras y gastos", "values": { "current": 41500, "reference": 39000, "difference": 2500 } },
    { "id": "payrolls", "kind": "category", "side": "expenses", "title": "Nóminas", "values": { "current": 22000, "reference": 21000, "difference": 1000 } },
    { "id": "expenses", "kind": "total", "title": "Total gastos", "values": { "current": 63500, "reference": 60000, "difference": 3500 } },
    { "id": "profit", "kind": "total", "title": "Beneficio", "values": { "current": 35700, "reference": 31500, "difference": 4200 } }
  ]
}
```

### Ejemplo: evolución mensual

```bash
curl "https://app.facturadirecta.com/api/$COMPANY_ID/reports/resultsSummary?startDate=2025-01-01&endDate=2025-12-31&breakdown=month&total=true" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Devuelve las columnas `p0` (enero) a `p11` (diciembre) y `total`.

## Listar los documentos de una categoría

`GET /{companyId}/reports/resultsDetail`

| Parámetro | Valores | Por defecto | Significado |
|---|---|---|---|
| `category` | una de las categorías | obligatorio | Categoría cuyos documentos se quieren ver. |
| `startDate`, `endDate` | fecha `YYYY-MM-DD` | año natural en curso | Periodo, con las dos fechas incluidas. Como máximo, 5 años. |
| `limit`, `offset` | ver [Paginación](../guides/pagination.md) | 25, 0 | Paginación por desplazamiento, hasta 500 por página. |

Con la categoría y las fechas de una columna del resumen, la suma de
`amount` de todas las páginas es el importe de esa celda.

La respuesta tiene `meta` (con `category`, `side` y `period`), `items` y
`pagination`. Cada elemento de `items`:

| Campo | Tipo | Significado |
|---|---|---|
| `id` | string | Documento que genera los apuntes (`inv_…`, `bil_…`, `par_…`, `tra_…`). |
| `type` | string | Tipo del documento (`invoice`, `bill`, `payroll`, `transaction`…), o `null` si no se reconoce. |
| `title` | string | Título del documento. |
| `date` | date | Fecha del primer apunte del documento en el periodo. |
| `amount` | number | Importe que el documento aporta a la categoría, con el signo de su lado. |

Los documentos van de más antiguo a más reciente. Con el `id` puedes
abrir el documento en su recurso, por ejemplo
[`GET /{companyId}/invoices/{id}`](./invoices.md).

### Ejemplo

```bash
curl "https://app.facturadirecta.com/api/$COMPANY_ID/reports/resultsDetail?category=currencyGains&startDate=2025-01-01&endDate=2025-12-31" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

```json
{
  "meta": {
    "report": "resultsDetail",
    "currency": "EUR",
    "category": "currencyGains",
    "side": "income",
    "period": { "start": "2025-01-01", "end": "2025-12-31" },
    "generatedAt": "2025-12-31T10:00:00.000Z"
  },
  "items": [
    { "id": "<id-cobro>", "type": "transaction", "title": "Cobro en dólares", "date": "2025-04-01", "amount": 1200 }
  ],
  "pagination": { "limit": 25, "offset": 0, "total": 1 }
}
```

## Obtener pérdidas y ganancias

`GET /{companyId}/reports/profitLoss`

| Parámetro | Valores | Por defecto | Significado |
|---|---|---|---|
| `startDate`, `endDate` | fecha `YYYY-MM-DD` | año natural en curso | Periodo, con las dos fechas incluidas. Como máximo, 5 años. |
| `compare`, `breakdown`, `difference`, `variance`, `total` | como en el resumen | | Mismas opciones de comparativa que el resumen de resultados. |
| `revenueShare` | booleano | `false` | Añade tras cada columna de periodo otra (`<id>_share`) con el peso de cada fila sobre la cifra de negocios. |
| `detail` | `summary`, `accounts`, `full` | `accounts` | Nivel de detalle: solo epígrafes, con grupos de cuentas o con cada cuenta y tercero. |
| `includeClosure` | booleano | `false` | Incluye la regularización y el cierre fechados el último día del periodo. |
| `excludeRevenueClearing` | booleano | `false` | Excluye la regularización de ingresos y gastos en todo el periodo. |
| `allTheseTags` | texto, repetible | | Solo los apuntes de documentos con todas estas etiquetas. |

### Ejemplo

```bash
curl "https://app.facturadirecta.com/api/$COMPANY_ID/reports/profitLoss?startDate=2025-01-01&endDate=2025-12-31&detail=summary&compare=previousYear" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

```json
{
  "meta": {
    "report": "profitLoss",
    "template": { "id": "profitLoss", "variant": "pyme", "version": 1 },
    "currency": "EUR",
    "detail": "summary",
    "policies": { "includeClosure": false, "excludeRevenueClearing": false },
    "columns": [
      { "id": "current", "kind": "movement", "start": "2025-01-01", "end": "2025-12-31" },
      { "id": "reference", "kind": "movement", "start": "2024-01-01", "end": "2024-12-31" }
    ],
    "generatedAt": "2025-12-31T10:00:00.000Z"
  },
  "rows": [
    { "id": "A_1", "parentId": "A", "kind": "accounts", "level": 2, "title": "1. Importe neto de la cifra de negocios", "values": { "current": 98000, "reference": 91500 } },
    { "id": "A_4", "parentId": "A", "kind": "accounts", "level": 2, "title": "4. Aprovisionamientos", "values": { "current": -41500, "reference": -39000 } },
    { "id": "A", "parentId": "C", "kind": "sum", "level": 1, "title": "A) RESULTADO DE EXPLOTACIÓN ( 1 + 2 + 3 + 4 + 5 + 6 + 7 + 8 + 9 + 10 + 11 )", "values": { "current": 56500, "reference": 52500 } },
    { "id": "D", "kind": "sum", "level": 1, "title": "D) RESULTADO DEL EJERCICIO (C + 17)", "values": { "current": 56500, "reference": 52500 } }
  ],
  "diagnostics": []
}
```

Los gastos aparecen en negativo, con el signo de presentación del modelo
oficial.

## Obtener el balance de situación

`GET /{companyId}/reports/balanceSheet`

| Parámetro | Valores | Por defecto | Significado |
|---|---|---|---|
| `date` | fecha `YYYY-MM-DD` | 31 de diciembre del año en curso | Fecha del saldo. |
| `startDate` | fecha `YYYY-MM-DD` | 1 de enero del año de `date` | Inicio del periodo para `breakdown` y `compare=previousPeriod`. Como máximo, 5 años antes de `date`. |
| `compare` | `none`, `previousPeriod`, `previousYear` | `none` | Saldo al final del periodo anterior (el día antes de `startDate`) o en la misma fecha del año anterior. |
| `breakdown` | `none`, `month`, `quarter`, `year` | `none` | Saldo al final de cada mes, trimestre o año del periodo, como máximo 24 columnas. |
| `difference`, `variance` | booleano | `false` | Con `compare`, añaden la diferencia y la variación. |
| `detail` | `summary`, `accounts`, `full` | `accounts` | Nivel de detalle, como en pérdidas y ganancias. |
| `includeClosure` | booleano | `false` | Incluye la regularización y el cierre fechados en la fecha del saldo. |
| `allTheseTags` | texto, repetible | | Solo los apuntes de documentos con todas estas etiquetas. |

El saldo de cada columna suma todos los apuntes hasta su fecha. El
resultado del ejercicio se calcula desde la cuenta de pérdidas y
ganancias, aunque el año no esté regularizado. Si el activo no coincide
con el patrimonio neto y el pasivo, aparece la fila `imbalance` con la
diferencia y el aviso `imbalance`.

Con `detail=full` el informe resuelve el tercero de todos los documentos
de la historia, y en empresas con mucho volumen puede tardar varios
segundos.

## Obtener sumas y saldos

`GET /{companyId}/reports/trialBalance`

| Parámetro | Valores | Por defecto | Significado |
|---|---|---|---|
| `startDate`, `endDate` | fecha `YYYY-MM-DD` | año natural en curso | Periodo. El saldo anterior es todo lo anterior a `startDate`, con el asiento de apertura de ese día. |
| `account` | cuenta o prefijo | | `43` incluye todas las cuentas de clientes y `430000` solo esa cuenta. |
| `groups` | `1`, `2`, `3`, repetible | | Añade subtotales por los primeros dígitos de la cuenta. |
| `thirdParty` | id de contacto, producto o banco | | Solo los apuntes de ese tercero, con el criterio del balance. |
| `thirdPartyDetail` | booleano | `false` | Añade bajo cada cuenta una fila por tercero. Necesita `thirdParty`. |
| `showCurrency` | booleano | `false` | Separa cada cuenta por moneda, con los importes en la moneda original en `currencyAmounts`. |
| `excludeBalance0` | booleano | `false` | Omite las filas con saldo final cero. |
| `includeClosure`, `excludeRevenueClearing` | booleano | `false` | Como en pérdidas y ganancias. |
| `allTheseTags` | texto, repetible | | Solo los apuntes de documentos con todas estas etiquetas. |
| `limit`, `offset` | ver [Paginación](../guides/pagination.md) | 25, 0 | La paginación se aplica a las filas ya agrupadas. |

`thirdPartyDetail` necesita `thirdParty` porque el detalle por tercero de
una cuenta entera, como la de clientes, puede tardar decenas de segundos
en empresas con miles de clientes.

### Ejemplo: saldo de un cliente

```bash
curl "https://app.facturadirecta.com/api/$COMPANY_ID/reports/trialBalance?startDate=2025-01-01&endDate=2025-12-31&account=430&thirdParty=<id-contacto>&thirdPartyDetail=true" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

```json
{
  "meta": {
    "report": "trialBalance",
    "currency": "EUR",
    "period": { "start": "2025-01-01", "end": "2025-12-31" },
    "groups": [],
    "thirdPartyDetail": true,
    "policies": { "includeClosure": false, "excludeRevenueClearing": false },
    "generatedAt": "2025-12-31T10:00:00.000Z"
  },
  "rows": [
    { "id": "430000", "kind": "account", "level": 0, "title": "430000 Clientes", "account": "430000", "accountTitle": "Clientes", "amounts": { "previousBalance": 0, "debit": 1210, "credit": 0, "balance": 1210 } },
    { "id": "430000/<id-contacto>", "kind": "thirdParty", "level": 1, "title": "Cliente Uno", "account": "430000", "accountTitle": "Clientes", "thirdParty": { "id": "<id-contacto>", "type": "contact", "name": "Cliente Uno" }, "amounts": { "previousBalance": 0, "debit": 1210, "credit": 0, "balance": 1210 } }
  ],
  "totals": { "previousBalance": 0, "debit": 1210, "credit": 0, "balance": 1210 },
  "pagination": { "limit": 25, "offset": 0, "total": 2 }
}
```

## Permisos

Todas las operaciones necesitan el scope `accounting:read`, el mismo que
el [diario](./journal.md): los informes muestran el beneficio y el gasto
en nóminas de la empresa.

Pérdidas y ganancias, el balance de situación y sumas y saldos necesitan
además el **módulo de Contabilidad**. Sin él responden `403` con
`plan_limit_exceeded` y `hint.features: ["fullAccounting"]`, sin enviar
ningún aviso a los administradores (ver
[Errores](../guides/errors.md#informes-no-incluidos-en-el-plan)). Una
aplicación conectada puede saber de antemano si una empresa lo tiene con
`accountingModule` en el [perfil](./profile.md).

## Errores comunes

- `400 Bad Request`: fechas no válidas, también las que no existen como
  `2026-02-30` (nunca se sustituyen por el periodo por defecto),
  `startDate` posterior a la fecha final, un periodo de más de 5 años,
  `compare` y `breakdown` a la vez,
  `difference`, `variance` o `total` sin la opción que necesitan, más de
  24 columnas o `thirdPartyDetail` sin `thirdParty`.
- `403 Forbidden`: la credencial no tiene el scope `accounting:read`, o
  la empresa no tiene el módulo de Contabilidad (`plan_limit_exceeded`).
- `422 Unprocessable Entity` (`report_query_timeout`): el informe ha
  tardado demasiado. Acota el periodo, el nivel de detalle o las columnas.

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| GET | `/{companyId}/reports/balanceSheet` | `getBalanceSheet` | `accounting:read` | Balance de situación |
| GET | `/{companyId}/reports/profitLoss` | `getProfitLoss` | `accounting:read` | Pérdidas y ganancias |
| GET | `/{companyId}/reports/resultsDetail` | `getResultsDetail` | `accounting:read` | Detalle del resumen de resultados |
| GET | `/{companyId}/reports/resultsSummary` | `getResultsSummary` | `accounting:read` | Resumen de resultados |
| GET | `/{companyId}/reports/trialBalance` | `getTrialBalance` | `accounting:read` | Sumas y saldos |

## Scopes

- **`accounting:read`** — Lectura de datos de contabilidad (diario).
