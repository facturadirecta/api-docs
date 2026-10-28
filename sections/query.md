---
title: Consulta analítica
audience:
  - developers
  - agents
status: draft
---

# Consulta analítica

La **consulta analítica** agrega los datos de la empresa en una sola
llamada: eliges una fuente, las medidas que quieres sumar, las dimensiones
por las que agrupar, un periodo y unos filtros, y la API devuelve una
tabla ya calculada. Es la forma de responder preguntas como «¿quiénes son
mis cinco mejores clientes del año?», «¿cuánto suman las facturas
vencidas?» o «¿qué provincias facturan más que el año pasado?» sin
recorrer los listados página a página.

Las cifras son las mismas que muestran los listados y los informes de la
aplicación: cada medida se define una sola vez y con el mismo criterio
que la pantalla equivalente. Los importes de las facturas son los del
documento (base, impuestos, total y pendiente); los del diario, los de
los apuntes.

Es de solo lectura. Las consultas se calculan en el momento, sobre los
datos actuales, y cada una tiene un tope de 30 segundos.

> Para los informes oficiales (resumen de resultados, pérdidas y
> ganancias, balance, sumas y saldos) usa los [informes](./reports.md):
> aplican las reglas de presentación de cada modelo. La consulta
> analítica sirve para rankings, totales y comparativas a medida.

## Fuentes

Cada fuente se llama como el recurso de la API cuyo listado devolvería
esos documentos; `invoiceLines` y `billLines` son las líneas de esos
mismos documentos, con su producto. No hay fuentes genéricas como
«ventas» o «compras»: la contabilidad es solo `journal`.

| Fuente | Qué contiene | Scope | Medida por defecto |
|---|---|---|---|
| `invoices` | [Facturas de venta](./invoices.md) definitivas, no anuladas y no borradas, con los importes del documento. Un apunte manual de ventas no está aquí: está en `journal`. | `invoices:read` | `totalBeforeTaxes` |
| `invoiceLines` | Líneas de las facturas de `invoices`, con su producto, su cantidad y su importe sin impuestos. Las líneas sin cantidad, como las de solo texto, no entran. Es la fuente de «¿qué producto vendo más?». | `invoices:read` | `totalBeforeTaxes` |
| `bills` | [Facturas de compra y tickets](./bills.md) definitivos, no anulados y no borrados, con los importes del documento. Las nóminas y las amortizaciones no están aquí: están en `journal`. | `bills:read` | `totalBeforeTaxes` |
| `billLines` | Líneas de las facturas de compra y tickets de `bills`, con su producto, su cantidad, su importe sin impuestos y su cuenta de gasto. Las líneas sin cantidad no entran. | `bills:read` | `totalBeforeTaxes` |
| `contacts` | [Contactos](./contacts.md) no borrados que la credencial puede ver: clientes, proveedores y empleados según sus permisos. | `contacts:read` | `count` |
| `journal` | Apuntes del [diario](./journal.md) de todos los documentos y asientos, sin los asientos de mantenimiento (apertura, regularización y cierre). Es la única fuente contable. | `accounting:read` | `net` |

Los borradores no entran salvo que lo pidas con el filtro `draft`, y los
documentos anulados, salvo que los pidas con el filtro `state`
(`voided` en `value` o entre los `values`). Los documentos borrados no
entran nunca.

En `invoices`, las facturas simplificadas (las de un destinatario sin
identificador fiscal) entran como las demás; el filtro `simplified` las
separa. Las sustitutivas de
simplificadas (F3) no entran nunca: repiten las ventas de las
simplificadas que sustituyen y no tienen asiento, así que contarlas
sumaría dos veces la misma venta. Los documentos con total cero entran
y cuentan en `count`.

## Petición

`POST /{companyId}/query` con un cuerpo JSON:

| Campo | Tipo | Significado |
|---|---|---|
| `from` | string | Fuente: `invoices`, `invoiceLines`, `bills`, `billLines`, `contacts` o `journal`. Obligatorio. |
| `measures` | string[] | Hasta 10 medidas de la fuente. En las medidas de días, la agregación va como sufijo: `daysToPayment.median`. Sin indicar, la medida por defecto de la fuente. |
| `dimensions` | string[] | Hasta 2 dimensiones por las que agrupar, en orden. Sin indicar, una sola fila con los totales. |
| `period` | object | `start` y `end` (`YYYY-MM-DD`), las dos fechas incluidas. Por defecto, el año natural en curso. Como máximo, 5 años. En `contacts` es opcional y sin tope: sin `period` entran todos los contactos. |
| `compare` | string | `previousPeriod` (el periodo anterior de la misma duración) o `previousYear` (el mismo periodo del año anterior). Añade a cada medida las columnas de referencia, diferencia y variación. |
| `breakdown` | string | `month`, `quarter` o `year`: divide el periodo en tramos naturales y añade la dimensión `period`. Como máximo, 24 tramos. No se combina con `compare`. |
| `filters` | object[] | Hasta 10 filtros `{ field, op, value }` o `{ field, op, values }`. Ver [Filtros](#filtros). |
| `orderBy` | object[] | Hasta 4 criterios `{ field, direction }` sobre las medidas y dimensiones de la consulta. `direction` es `asc` o `desc` (por defecto). Sin indicar, la primera medida de mayor a menor. |
| `limit` | integer | Filas como máximo, entre 1 y 500. Por defecto, 10. |
| `limitPerGroup` | integer | Con dos dimensiones, como máximo estas filas de la segunda dentro de cada valor de la primera: las primeras según las medidas de `orderBy` (o la primera medida, de mayor a menor). «Los cinco productos más vendidos de cada cliente». No se combina con `compare`, `breakdown` ni `includeOther`. |
| `includeOther` | boolean | Añade `other` con la suma de los grupos que quedan fuera de `limit`. No se aplica con `breakdown`. |
| `amounts` | string | Moneda de los importes en `invoices`, `bills` y sus líneas: `document` (por defecto), cada importe en la de su documento, con la moneda como dimensión implícita si hay varias; `company`, el contravalor de cada documento en la moneda de la empresa (`meta.currency`), en un solo grupo aunque haya varias monedas. `pending` solo admite `document`; con multimoneda, `cost`, `margin` y `marginRate` solo admiten `company`. |

El periodo acota por la fecha del documento en `invoices` y `bills` y por
la fecha del apunte en `journal`. En `contacts` acota por la fecha de alta
del contacto y es opcional: sin `period`, la consulta cuenta todos los
contactos («¿cuántos clientes tengo por provincia?»); con `period`, los
dados de alta en esas fechas, sin el tope de 5 años. `compare` y
`breakdown` necesitan `period` también en `contacts`.

## Medidas

### Facturas de venta y de compra

| Medida | Tipo | Significado |
|---|---|---|
| `count` | número | Documentos del conjunto filtrado. |
| `contacts` | número | Contactos distintos de los documentos del grupo. Con la dimensión `fiscalId`, más de uno indica contactos duplicados con el mismo identificador fiscal. |
| `totalBeforeTaxes` | importe | Suma de la base imponible de cada documento, en su moneda. Es la cifra que el listado muestra como base. |
| `taxes` | importe | Suma de la diferencia entre el total y la base imponible: IVA, recargos y retenciones con su signo. |
| `indirectTaxes` | importe | Suma de los impuestos sin las retenciones de IRPF: IVA, recargo de equivalencia, IGIC o IPSI. Es la columna «Impuestos» de los informes «Ventas por cliente» y «Compras por proveedor». |
| `retentions` | importe | Suma de las retenciones de IRPF, en positivo. |
| `gross` | importe | Base imponible más los impuestos sin descontar las retenciones (`totalBeforeTaxes` + `indirectTaxes`): el «Importe total» de los informes «Ventas por cliente» y «Compras por proveedor». |
| `total` | importe | Suma del total de cada documento, impuestos incluidos. |
| `pending` | importe | Importe pendiente de cobro (o de pago, en `bills`): el total menos lo cobrado. Negativo si se ha cobrado de más. |
| `average` | importe | Base imponible media por documento: `totalBeforeTaxes` entre `count`. |
| `daysToPayment` | días | Días entre la fecha del documento y la fecha del último cobro (o pago) que lo salda. Solo cuentan los documentos cobrados. Cobros y pagos a fondo: en todos los planes de pago, no en el Gratis. |
| `daysOverdue` | días | Días entre el vencimiento y el cobro, o hasta hoy si sigue pendiente; nunca negativos. Solo cuentan los documentos con vencimiento. Cobros y pagos a fondo: en todos los planes de pago, no en el Gratis. |

Las medidas de días admiten la agregación como sufijo: `.avg` (por
defecto), `.median`, `.p90`, `.min`, `.max` o `.count` (cuántos
documentos entran en el cálculo). Puedes pedir varias a la vez:
`daysToPayment` y `daysToPayment.median`.

### Líneas de facturas de venta y de compra

En `invoiceLines` y `billLines`:

| Medida | Tipo | Significado |
|---|---|---|
| `count` | número | Líneas del conjunto filtrado. |
| `documents` | número | Documentos distintos con alguna línea en el grupo. |
| `contacts` | número | Contactos distintos de los documentos del grupo: por ejemplo, cuántos clientes compran un producto. |
| `quantity` | número | Suma de las cantidades de las líneas, con hasta cuatro decimales. |
| `totalBeforeTaxes` | importe | Suma del importe de las líneas sin impuestos, con el descuento de cada línea, en la moneda del documento. |
| `averagePrice` | importe | `totalBeforeTaxes` entre `quantity`: el precio medio por unidad. |

En `invoiceLines`, además, el margen de las ventas, incluido en todos los planes de pago
(ver [Planes](#planes)). El coste sale del precio de compra que tiene hoy el
producto, en la moneda de la empresa, así que estas medidas van siempre en esa
moneda: con multimoneda, piden `amounts: "company"` para que las ventas también
lo estén (si no, `400`).

| Medida | Tipo | Significado |
|---|---|---|
| `cost` | importe | Cantidad por el precio de compra del producto (el de su faceta de compra, con su descuento). Las líneas sin producto, o con un producto sin precio de compra, no tienen coste: un grupo sin ninguna con coste da `null`. |
| `margin` | importe | `totalBeforeTaxes` menos `cost`, solo de las líneas con coste. |
| `marginRate` | ratio | `margin` entre la base imponible de las líneas con coste, como fracción (`0.25` es un 25 %). |

### Contactos

| Medida | Tipo | Significado |
|---|---|---|
| `count` | número | Contactos del conjunto filtrado. |

### Diario

Todos los importes van en la moneda de la empresa.

| Medida | Tipo | Significado |
|---|---|---|
| `count` | número | Apuntes del conjunto filtrado. |
| `debit` | importe | Suma del debe. |
| `credit` | importe | Suma del haber. |
| `balance` | importe | Debe menos haber: saldo deudor en positivo. |
| `net` | importe | Importe con el signo natural de cada cuenta: haber menos debe en el grupo 7 (ingresos en positivo) y debe menos haber en el resto (gastos en positivo). Es la medida de un ranking de ventas o de gastos. |
| `income` | importe | Haber menos debe de los apuntes del grupo 7, como el resumen de resultados. |
| `expenses` | importe | Debe menos haber de los apuntes del grupo 6, como el resumen de resultados. |
| `result` | importe | `income` menos `expenses`. |

## Dimensiones

### Facturas de venta y de compra

| Dimensión | Valor | Significado |
|---|---|---|
| `contact` | entidad | Cliente o proveedor del documento. |
| `contact.region` | entidad | Provincia de la dirección actual del contacto (la del contacto hoy, no la del documento). `id` es el código de provincia y `name` su nombre, para España. |
| `contact.country` | código | País de la dirección actual del contacto (código ISO de dos letras), no el del documento. |
| `fiscalId` | entidad | Identificador fiscal de la contraparte: NIF, CIF o NIE en España, NIF-IVA intracomunitario (VAT number) en la UE o el identificador fiscal de su país. `id` es el identificador normalizado (ver [Identificador fiscal](#identificador-fiscal)) y `name` el nombre de la contraparte en el documento más reciente del grupo. |
| `country` | código | País del documento (ISO 3166-1 alpha-2). |
| `series` | código | Serie de numeración. |
| `currency` | código | Moneda del documento. |
| `state` | valor | Estado del documento: `pending`, `overdue`, `paid`, `overpaid` o `voided`. |
| `paymentMethod` | entidad | Método de cobro o de pago del documento. |
| `agingBucket` | tramo | Antigüedad del pendiente: `notDue` (sin vencer), `1-30`, `31-60`, `61-90` y `over90` días de retraso; `settled` para los cobrados o pagados. Cobros y pagos a fondo: en todos los planes de pago, no en el Gratis. |
| `day`, `week`, `month`, `quarter`, `year` | periodo | Día, semana ISO (de lunes a domingo), mes, trimestre o año natural de la fecha del documento (`2026-03-31`, `2026-W14`, `2026-03`, `2026-Q1`, `2026`). |
| `account` | código | Solo en `bills`: cuenta contable de gasto de la cabecera del documento. |

### Líneas de facturas de venta y de compra

En `invoiceLines` y `billLines`, además de `contact`, `contact.region`,
`contact.country`, `fiscalId`, `country`, `series`, `currency` y las de
tiempo, que son las del documento de la línea:

| Dimensión | Valor | Significado |
|---|---|---|
| `product` | entidad | Producto o servicio de la línea, con `id`, `name` y `sku` (su referencia). Las líneas sin producto forman el grupo `null`. |
| `account` | código | Cuenta contable de la línea: de ventas en `invoiceLines`, de gasto en `billLines`. |

### Contactos

| Dimensión | Valor | Significado |
|---|---|---|
| `address.region` | entidad | Provincia de la dirección del contacto, con `id` (código) y `name` (nombre, para España). |
| `address.country` | código | País de la dirección del contacto. |
| `fiscalId` | entidad | Identificador fiscal del contacto, normalizado como en las facturas. Con `count`, un grupo de más de uno son contactos duplicados. `name` es el nombre del contacto dado de alta más recientemente. |
| `isClient`, `isProvider`, `isEmployee` | booleano | El contacto tiene esa faceta. |
| `createdMonth` | periodo | Mes natural de alta del contacto (`2026-03`). |

### Diario

| Dimensión | Valor | Significado |
|---|---|---|
| `account` | entidad | Cuenta contable completa, con `id` (`700000`) y `name` (su título en el plan contable). |
| `account.group`, `account.subgroup`, `account.code` | código | La primera cifra de la cuenta (`7`), las dos primeras (`70`) o las tres primeras (`700`). |
| `thirdParty` | entidad | A quién se imputa el apunte: el contacto en las cuentas de clientes y proveedores, el banco en las de tesorería, el producto en las de ventas y compras cuando la línea lleva producto. Incluye `type` (`contact`, `bank` o `product`). |
| `contact` | entidad | Contacto del documento que generó el asiento (la factura, la compra, la nómina, el cobro o la cabecera de un asiento manual). Si el documento no tiene contacto y el asiento tiene una sola contrapartida de cliente, proveedor o acreedor (cuentas 43, 40 o 41), ese contacto. Si no, `unassigned`. |
| `category` | valor | Origen del importe, el del [resumen de resultados](./reports.md): `invoices`, `currencyGains`, `otherIncome`, `bills`, `payrolls`, `depreciation`, `currencyLosses`, `otherExpenses`. Solo los apuntes de los grupos 6 y 7 tienen categoría. |
| `documentType` | valor | Tipo del documento que generó el asiento: `invoice`, `bill`, `payroll`, `transaction` (cobros, pagos y asientos manuales) u `other`. |
| `day`, `week`, `month`, `quarter`, `year` | periodo | Día, semana ISO, mes, trimestre o año natural de la fecha del apunte. |

La dimensión `contact` del diario es la que responde «ventas de las
cuentas 7 por cliente» incluyendo los asientos manuales, algo que
`invoices` no ve. Un apunte de ventas sin contacto queda en `unassigned`.

### Identificador fiscal

`fiscalId` agrupa por el identificador fiscal y no por el contacto: dos
contactos duplicados con el mismo NIF suman en una sola fila. En las
facturas es el identificador que el documento copió de su contacto al
emitirse; los documentos más antiguos, que no lo guardan, toman el del
contacto hoy, como el listado.

Para agrupar, el identificador se normaliza según su país:

| País del identificador | Clave | Ejemplo |
|---|---|---|
| España | En mayúsculas, sin espacios ni signos, sin el prefijo `ES` y con el DNI completado con ceros a la izquierda. | `ESB-12345674` y `b12345674` son `B12345674`; `1234567L` es `01234567L`. |
| Resto de la UE | En mayúsculas, sin espacios ni signos y en la forma del NIF-IVA intracomunitario: con el prefijo de su país, lo lleve o no (`EL` en Grecia). | `12 345 678 901` de Francia es `FR12345678901`. |
| Resto del mundo | En mayúsculas y sin espacios ni signos. | `12-3456789` es `123456789`. |

El país es el del identificador del contacto o, si no lo tiene, el de su
dirección. Los documentos sin identificador forman el grupo `null`, que
en un negocio con muchos tickets puede ser el más grande.

El filtro `fiscalId` busca la clave tal como la da la dimensión. Admite
separadores y minúsculas, y un NIF español con el prefijo `ES` o un DNI
sin sus ceros. Un identificador de otro país de la UE tiene que llevar su
prefijo (`FR12345678901`): sin él, las mismas cifras podrían ser de otro
país.

## Filtros

Cada filtro tiene un campo, un operador y un valor (`value`) o una lista
de valores (`values`):

| Operador | Valor | Uso |
|---|---|---|
| `eq` | `value` | Igual a. |
| `in` | `values` | Cualquiera de la lista, hasta 100 valores. |
| `gte`, `lte` | `value` | Mayor o igual, menor o igual. Importes y fechas. |
| `between` | `values` | Dos valores, los dos incluidos. Importes y fechas. |
| `startsWith` | `value` | Empieza por. Cuentas contables: `7`, `629`. |

| Fuente | Filtro | Operadores | Significado |
|---|---|---|---|
| `invoices`, `bills` | `contact` | `eq`, `in` | Identificador del contacto (`con_…`). |
| | `contact.region` | `eq`, `in` | Código de provincia de la dirección del contacto. |
| | `contact.country` | `eq`, `in` | Código ISO del país de la dirección del contacto. |
| | `fiscalId` | `eq`, `in` | Identificador fiscal de la contraparte, sea cual sea el contacto. Ver [Identificador fiscal](#identificador-fiscal). |
| | `country` | `eq`, `in` | Código ISO del país del documento. |
| | `series` | `eq`, `in` | Serie de numeración. |
| | `currency` | `eq`, `in` | Moneda del documento. |
| | `state` | `eq`, `in` | `pending`, `overdue`, `paid`, `overpaid` o `voided`. Sin este filtro, los anulados no entran. |
| | `paymentMethod` | `eq`, `in` | Identificador del método de pago. |
| | `total`, `totalBeforeTaxes` | `gte`, `lte`, `between` | Importes del documento. |
| | `dueDate` | `gte`, `lte`, `between` | Fecha de vencimiento (`YYYY-MM-DD`). |
| | `tag` | `eq`, `in` | Etiquetas que el documento debe tener, todas ellas. |
| | `draft` | `eq` | `true` incluye los borradores junto a los definitivos. |
| `invoices` | `simplified` | `eq` | `true` deja solo las facturas simplificadas; `false` las deja fuera. Sin este filtro entran las dos. |
| `bills` | `account` | `eq`, `in`, `startsWith` | Cuenta contable de gasto del documento. |
| `invoiceLines`, `billLines` | `product` | `eq`, `in` | Identificador del producto (`pro_…`). |
| | `hasProduct` | `eq` | `true` deja solo las líneas con producto; `false`, solo las de texto libre. |
| | `account` | `eq`, `in`, `startsWith` | Cuenta contable de la línea. |
| | `contact`, `contact.region`, `contact.country`, `fiscalId`, `country`, `series`, `currency`, `tag`, `draft` | | Los del documento de la línea, como en `invoices` y `bills`. |
| `invoiceLines` | `simplified` | `eq` | Las líneas de facturas simplificadas, solas (`true`) o fuera (`false`). |
| `contacts` | `address.region`, `address.country` | `eq`, `in` | Provincia y país de la dirección. |
| | `fiscalId` | `eq`, `in` | Identificador fiscal del contacto. |
| | `isClient`, `isProvider`, `isEmployee` | `eq` | Faceta del contacto. |
| | `tag` | `eq`, `in` | Etiquetas que el contacto debe tener, todas ellas. |
| `journal` | `account` | `eq`, `in`, `startsWith` | Cuenta contable o prefijo. |
| | `thirdParty` | `eq`, `in` | Identificador del contacto, banco o producto al que se imputa el apunte. |
| | `contact` | `eq`, `in` | Identificador del contacto del documento que generó el asiento. |
| | `category` | `eq`, `in` | Categoría del resumen de resultados. |
| | `documentType` | `eq`, `in` | Tipo del documento que generó el asiento. |

## Respuesta

| Campo | Tipo | Significado |
|---|---|---|
| `meta.source`, `meta.basis` | string | La fuente y su base: `documents` (cifras de los documentos) o `accounting` (cifras del diario). |
| `meta.period` | object | El periodo consultado, `start` y `end`. Ausente en `contacts` sin `period`. |
| `meta.compare` | object | Con `compare`: `kind`, `start` y `end` del periodo de referencia. |
| `meta.breakdown` | string | Con `breakdown`: la unidad de los tramos. |
| `meta.currency` | string | Moneda de los importes cuando `currency` no es una dimensión de la respuesta o cuando van en la de la empresa (`amounts: "company"`). |
| `meta.limit`, `meta.generatedAt` | | El límite aplicado y el momento del cálculo. |
| `columns` | object[] | Qué es cada clave de las filas, en orden: `id`, `kind` (`dimension` o `measure`), `title`, `description`, `type` y, en las medidas, `measure` y `period`. |
| `rows` | object[] | Los grupos, ordenados y limitados. Una clave por columna. |
| `totals` | object | El valor de cada medida sobre el conjunto filtrado completo, no solo sobre las filas devueltas. Con `compare`, también el del periodo de referencia, la diferencia y la variación (`<medida>_reference`, `_difference` y `_variance`). |
| `other` | object | Con `includeOther`, si hay más grupos que `limit`: `groups` (cuántos quedan fuera) y la suma de cada medida que se puede sumar; `null` en las que no (`average`, las de días). Con `compare`, también `<medida>_reference`, `_difference` y `_variance`; la referencia del resto es el total de referencia menos el de las filas, así que la tabla cuadra con `totals`. |
| `truncated` | boolean | Había más grupos que `limit`. |

En cada fila, el valor de una dimensión depende de su tipo:

- **Entidad** (`contact`, `product`, `fiscalId`, `paymentMethod`, `thirdParty`, `account`, las
  provincias): `{ "id", "name" }`, más `type` en `thirdParty`, `fiscalId` y
  `country` (los del contacto hoy) en `contact`, `sku` en `product`, y `webUrl`
  cuando el elemento tiene ficha en la aplicación (ver
  [Abrir un resultado en FacturaDirecta](../guides/overview.md#abrir-un-resultado-en-facturadirecta)).
  `name` es `null` si el elemento ya no existe.
- **Periodo**: las dimensiones de tiempo (`day`, `week`, `month`, `quarter`,
  `year`, `createdMonth`) valen un texto (`2026-W14`, `2026-03`); la dimensión
  `period` del desglose, `{ "id", "start", "end" }`.
- El resto: un texto, un número o un booleano. Una dimensión sin valor
  (un documento sin país, por ejemplo) vale `null` y forma su propio
  grupo.

Las medidas valen un número, o `null` cuando no hay dato (la media de
días de un cliente sin cobros). Los importes van redondeados a céntimos
y los días a una décima.

Con `compare`, cada medida añade tres columnas: `<medida>_reference` (el
valor en el periodo de referencia), `<medida>_difference` (periodo menos
referencia) y `<medida>_variance` (la variación como fracción, `0.2` es
un 20 %, o `null` si la referencia es cero). Con `breakdown`, la columna
`period` identifica el tramo de cada fila.

En empresas con multimoneda, `currency` es una dimensión implícita de
`invoices`, `bills` y sus líneas cuando la consulta pide importes:
aparece en las filas aunque no la pidas y los importes de monedas
distintas nunca se mezclan. Una consulta sin importes (documentos,
cantidades, días) no se separa por moneda. Con `amounts: "company"` los
importes van en la moneda de la empresa y tampoco se separan. Los
importes de `journal` van siempre en la moneda de la empresa.

## Operaciones

- [Consultar](#consultar)

## Consultar

`POST /{companyId}/query`

Necesita el scope de la fuente: `invoices:read`, `bills:read`,
`contacts:read` o `accounting:read`. Una API key con varios de ellos
puede consultar varias fuentes, una por llamada. No admite la cabecera
`Idempotency-Key`: repetir la consulta no tiene efectos.

### Ejemplo: los cinco mejores clientes del año

```bash
curl "https://app.facturadirecta.com/api/$COMPANY_ID/query" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "invoices",
    "measures": ["totalBeforeTaxes", "count"],
    "dimensions": ["contact"],
    "limit": 5,
    "includeOther": true
  }'
```

```json
{
  "meta": {
    "source": "invoices",
    "basis": "documents",
    "period": { "start": "2026-01-01", "end": "2026-12-31" },
    "currency": "EUR",
    "limit": 5,
    "generatedAt": "2026-10-18T10:00:00.000Z"
  },
  "columns": [
    { "id": "contact", "kind": "dimension", "title": "Contacto", "description": "Cliente o proveedor del documento (`id`, `name`).", "type": "entity" },
    { "id": "totalBeforeTaxes", "kind": "measure", "title": "Base imponible", "description": "Suma de la base imponible de cada factura, en la moneda del documento. Es la cifra que el listado muestra como «base».", "type": "money", "measure": "totalBeforeTaxes", "period": { "start": "2026-01-01", "end": "2026-12-31" } },
    { "id": "count", "kind": "measure", "title": "Documentos", "description": "Número de facturas del conjunto filtrado.", "type": "count", "measure": "count", "period": { "start": "2026-01-01", "end": "2026-12-31" } }
  ],
  "rows": [
    { "contact": { "id": "<id-contacto>", "name": "Cliente Uno", "webUrl": "https://app.facturadirecta.com/company/<id-empresa>/contacts/<id-contacto>" }, "totalBeforeTaxes": 18400, "count": 14 },
    { "contact": { "id": "<id-contacto-2>", "name": "Cliente Dos", "webUrl": "https://app.facturadirecta.com/company/<id-empresa>/contacts/<id-contacto-2>" }, "totalBeforeTaxes": 12750, "count": 9 }
  ],
  "totals": { "totalBeforeTaxes": 48200, "count": 61 },
  "other": { "groups": 23, "totalBeforeTaxes": 17050, "count": 38 },
  "truncated": true
}
```

La respuesta real tiene cinco filas; aquí se muestran dos. `totals` suma
las 61 facturas del año y `other` las 38 de los 23 clientes que quedan
fuera de las cinco primeras filas.

### Ejemplo: los mejores clientes por identificador fiscal

Si el mismo cliente está dado de alta dos veces, `contact` lo reparte en
dos filas. `fiscalId` lo reúne, y `contacts` dice cuántos contactos hay
detrás de cada fila.

```bash
curl "https://app.facturadirecta.com/api/$COMPANY_ID/query" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "invoices",
    "measures": ["totalBeforeTaxes", "contacts"],
    "dimensions": ["fiscalId"],
    "limit": 5
  }'
```

```json
{
  "rows": [
    { "fiscalId": { "id": "B12345674", "name": "Cliente Uno SL" }, "totalBeforeTaxes": 21300, "contacts": 2 },
    { "fiscalId": { "id": "FR12345678901", "name": "Client Deux SARL" }, "totalBeforeTaxes": 9800, "contacts": 1 }
  ],
  "totals": { "totalBeforeTaxes": 48200, "contacts": 37 }
}
```

La primera fila reúne dos contactos con el NIF `B12345674`. Para ver
cuáles son, repite la consulta con `"dimensions": ["fiscalId", "contact"]`
y el filtro `{ "field": "fiscalId", "op": "eq", "value": "B12345674" }`.
Para buscar todos los contactos duplicados sin mirar facturas, consulta
`contacts` con la medida `count` y la dimensión `fiscalId`: los grupos de
más de uno salen primero.

### Ejemplo: facturas vencidas y cuánto suman

Sin dimensiones, la respuesta es una sola fila con los totales del
conjunto filtrado.

```bash
curl "https://app.facturadirecta.com/api/$COMPANY_ID/query" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "invoices",
    "measures": ["count", "total", "pending"],
    "period": { "start": "2022-01-01", "end": "2026-12-31" },
    "filters": [{ "field": "state", "op": "in", "values": ["overdue"] }]
  }'
```

```json
{
  "meta": { "source": "invoices", "basis": "documents", "period": { "start": "2022-01-01", "end": "2026-12-31" }, "currency": "EUR", "limit": 10, "generatedAt": "2026-10-18T10:00:00.000Z" },
  "columns": [
    { "id": "count", "kind": "measure", "title": "Documentos", "description": "Número de facturas del conjunto filtrado.", "type": "count", "measure": "count", "period": { "start": "2022-01-01", "end": "2026-12-31" } },
    { "id": "total", "kind": "measure", "title": "Total", "description": "Suma del total de cada factura, impuestos incluidos, en la moneda del documento.", "type": "money", "measure": "total", "period": { "start": "2022-01-01", "end": "2026-12-31" } },
    { "id": "pending", "kind": "measure", "title": "Pendiente", "description": "Importe pendiente de cobro, en la moneda del documento: el total menos lo cobrado; negativo si se ha cobrado de más.", "type": "money", "measure": "pending", "period": { "start": "2022-01-01", "end": "2026-12-31" } }
  ],
  "rows": [
    { "count": 3, "total": 1815, "pending": 1815 }
  ],
  "totals": { "count": 3, "total": 1815, "pending": 1815 },
  "truncated": false
}
```

Para la lista de esas facturas, con sus datos, usa el
[listado](./invoices.md#lista-de-facturas) con `state=overdue`; con
`totals=true` el listado devuelve también estas sumas.

### Ejemplo: provincias con más facturación, frente al año anterior

```bash
curl "https://app.facturadirecta.com/api/$COMPANY_ID/query" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "invoices",
    "measures": ["totalBeforeTaxes"],
    "dimensions": ["contact.region"],
    "period": { "start": "2026-01-01", "end": "2026-06-30" },
    "compare": "previousYear",
    "limit": 5
  }'
```

```json
{
  "meta": { "source": "invoices", "basis": "documents", "period": { "start": "2026-01-01", "end": "2026-06-30" }, "compare": { "kind": "previousYear", "start": "2025-01-01", "end": "2025-06-30" }, "currency": "EUR", "limit": 5, "generatedAt": "2026-10-18T10:00:00.000Z" },
  "columns": [
    { "id": "contact.region", "kind": "dimension", "title": "Provincia del contacto", "description": "Provincia de la dirección actual del contacto (la del contacto hoy, no la del documento): `id` es el código y `name` el nombre para España.", "type": "entity" },
    { "id": "totalBeforeTaxes", "kind": "measure", "title": "Base imponible", "description": "…", "type": "money", "measure": "totalBeforeTaxes", "period": { "start": "2026-01-01", "end": "2026-06-30" } },
    { "id": "totalBeforeTaxes_reference", "kind": "measure", "title": "Base imponible (referencia)", "description": "La medida en el periodo de referencia (el mismo periodo del año anterior).", "type": "money", "measure": "totalBeforeTaxes", "period": { "start": "2025-01-01", "end": "2025-06-30" } },
    { "id": "totalBeforeTaxes_difference", "kind": "measure", "title": "Base imponible (diferencia)", "description": "Periodo menos referencia.", "type": "money", "measure": "totalBeforeTaxes" },
    { "id": "totalBeforeTaxes_variance", "kind": "measure", "title": "Base imponible (variación)", "description": "Variación sobre la referencia, como fracción (`0.2` es un 20 %); `null` si la referencia es cero.", "type": "ratio", "measure": "totalBeforeTaxes" }
  ],
  "rows": [
    { "contact.region": { "id": "08", "name": "Barcelona" }, "totalBeforeTaxes": 31200, "totalBeforeTaxes_reference": 27400, "totalBeforeTaxes_difference": 3800, "totalBeforeTaxes_variance": 0.1387 },
    { "contact.region": { "id": "28", "name": "Madrid" }, "totalBeforeTaxes": 12900, "totalBeforeTaxes_reference": 14100, "totalBeforeTaxes_difference": -1200, "totalBeforeTaxes_variance": -0.0851 }
  ],
  "totals": { "totalBeforeTaxes": 52300, "totalBeforeTaxes_reference": 48900, "totalBeforeTaxes_difference": 3400, "totalBeforeTaxes_variance": 0.0695 },
  "truncated": false
}
```

### Ejemplo: ventas de las cuentas 7 por cliente, asientos manuales incluidos

```bash
curl "https://app.facturadirecta.com/api/$COMPANY_ID/query" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "journal",
    "measures": ["net"],
    "dimensions": ["contact"],
    "filters": [{ "field": "account", "op": "startsWith", "value": "7" }],
    "limit": 5
  }'
```

```json
{
  "meta": { "source": "journal", "basis": "accounting", "period": { "start": "2026-01-01", "end": "2026-12-31" }, "currency": "EUR", "limit": 5, "generatedAt": "2026-10-18T10:00:00.000Z" },
  "columns": [
    { "id": "contact", "kind": "dimension", "title": "Contacto del asiento", "description": "…", "type": "entity" },
    { "id": "net", "kind": "measure", "title": "Importe neto", "description": "…", "type": "money", "measure": "net", "period": { "start": "2026-01-01", "end": "2026-12-31" } }
  ],
  "rows": [
    { "contact": { "id": "<id-contacto>", "name": "Cliente Uno", "webUrl": "https://app.facturadirecta.com/company/<id-empresa>/contacts/<id-contacto>" }, "net": 18400 },
    { "contact": { "id": "unassigned", "name": null }, "net": 300 }
  ],
  "totals": { "net": 48500 },
  "truncated": false
}
```

### Ejemplo: resultado del año mes a mes

```bash
curl "https://app.facturadirecta.com/api/$COMPANY_ID/query" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "journal",
    "measures": ["income", "expenses", "result"],
    "breakdown": "month",
    "limit": 12
  }'
```

Cada fila lleva la dimensión `period` con el tramo:

```json
{
  "meta": { "source": "journal", "basis": "accounting", "period": { "start": "2026-01-01", "end": "2026-12-31" }, "breakdown": "month", "currency": "EUR", "limit": 12, "generatedAt": "2026-10-18T10:00:00.000Z" },
  "columns": [
    { "id": "period", "kind": "dimension", "title": "Periodo", "description": "Tramo del desglose por meses: `id`, `start` y `end`.", "type": "period" },
    { "id": "income", "kind": "measure", "title": "Ingresos", "description": "…", "type": "money", "measure": "income", "period": { "start": "2026-01-01", "end": "2026-12-31" } },
    { "id": "expenses", "kind": "measure", "title": "Gastos", "description": "…", "type": "money", "measure": "expenses", "period": { "start": "2026-01-01", "end": "2026-12-31" } },
    { "id": "result", "kind": "measure", "title": "Resultado", "description": "…", "type": "money", "measure": "result", "period": { "start": "2026-01-01", "end": "2026-12-31" } }
  ],
  "rows": [
    { "period": { "id": "2026-01", "start": "2026-01-01", "end": "2026-01-31" }, "income": 8200, "expenses": 5100, "result": 3100 },
    { "period": { "id": "2026-02", "start": "2026-02-01", "end": "2026-02-28" }, "income": 7600, "expenses": 5400, "result": 2200 }
  ],
  "totals": { "income": 91500, "expenses": 60300, "result": 31200 },
  "truncated": false
}
```

Las filas del desglose van por tramo y, dentro de cada tramo, por el
orden pedido. Un tramo sin apuntes no aparece.

### Ejemplo: días medios de cobro por cliente

```bash
curl "https://app.facturadirecta.com/api/$COMPANY_ID/query" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "invoices",
    "measures": ["daysToPayment", "daysToPayment.median", "count"],
    "dimensions": ["contact"],
    "orderBy": [{ "field": "daysToPayment", "direction": "desc" }],
    "limit": 10
  }'
```

```json
{
  "rows": [
    { "contact": { "id": "<id-contacto>", "name": "Cliente Uno", "webUrl": "…" }, "daysToPayment": 41.5, "daysToPayment.median": 38, "count": 14 },
    { "contact": { "id": "<id-contacto-2>", "name": "Cliente Dos", "webUrl": "…" }, "daysToPayment": null, "daysToPayment.median": null, "count": 2 }
  ],
  "totals": { "daysToPayment": 27.3, "daysToPayment.median": 24, "count": 61 }
}
```

El segundo cliente tiene dos facturas sin cobrar: cuenta en `count`, pero
no en los días, que valen `null`.

### Ejemplo: los productos que más vendes a cada cliente

Las tres primeras líneas de producto de cada cliente, por unidades, sin
las líneas de texto libre:

```bash
curl "https://app.facturadirecta.com/api/$COMPANY_ID/query" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "invoiceLines",
    "measures": ["quantity", "totalBeforeTaxes"],
    "dimensions": ["contact", "product"],
    "filters": [{ "field": "hasProduct", "op": "eq", "value": true }],
    "orderBy": [{ "field": "contact", "direction": "asc" }, { "field": "quantity", "direction": "desc" }],
    "limitPerGroup": 3,
    "limit": 100
  }'
```

```json
{
  "rows": [
    { "contact": { "id": "<id-contacto>", "name": "Cliente Uno", "webUrl": "…" }, "product": { "id": "<id-producto>", "name": "Tornillo M4", "sku": "TOR-M4", "webUrl": "…" }, "quantity": 1200, "totalBeforeTaxes": 240 },
    { "contact": { "id": "<id-contacto>", "name": "Cliente Uno", "webUrl": "…" }, "product": { "id": "<id-producto-2>", "name": "Tuerca M4", "sku": "TUE-M4", "webUrl": "…" }, "quantity": 800, "totalBeforeTaxes": 96 }
  ],
  "totals": { "quantity": 5310, "totalBeforeTaxes": 1874.5 }
}
```

`totals` es el de todas las líneas del filtro, no solo el de las filas
que deja `limitPerGroup`.

## Qué cifras salen

- Una consulta sin dimensiones sobre `invoices` da el mismo recuento y
  las mismas sumas que el [listado de facturas](./invoices.md) con los
  mismos filtros, `state` con todos los estados menos `draft` y `voided`,
  y `totals=true`, salvo por las sustitutivas (F3), que el listado sí
  incluye. Lo mismo entre `bills` y el
  [listado de facturas de compra](./bills.md), que no tiene sustitutivas.
- `invoiceLines` agrupado por `product` da las mismas cantidades e
  importes que el informe «Ventas por producto» de la aplicación, y
  `billLines`, que «Compras por producto», salvo que este cuenta también
  las líneas de las compras anuladas.
- Una factura anulada tiene los importes a cero: pedirla con `state`
  solo cambia `count` y las medidas que dependen de él (`average`,
  `contacts`).
- `journal` con la dimensión `category` da las mismas cifras que el
  [resumen de resultados](./reports.md#obtener-el-resumen-de-resultados):
  los apuntes de los grupos 6 y 7 del periodo, sin el asiento de
  regularización.
- La provincia (`contact.region`) es la de la dirección actual del
  contacto. Un cliente que cambió de provincia cuenta en la nueva para
  todas sus facturas.
- `state` y `agingBucket` se calculan a la fecha de hoy en la zona
  horaria de la empresa, como el listado.
- Las medidas que no se suman (`average`, `contacts` y las de días) se
  calculan en `totals` sobre todos los documentos, no como media o suma
  de las filas.

## Permisos

Cada fuente necesita su scope: `invoices:read` (también para
`invoiceLines`), `bills:read` (también para `billLines`),
`contacts:read` o `accounting:read`. En `contacts`, la credencial solo
ve las facetas (clientes, proveedores, empleados) que su permiso le da.

## Planes

Todos los planes consultan sus datos. Dos grupos de nombres están incluidos
en todos los planes de pago y no en el Gratis, igual en la aplicación, en la
API y en el MCP:

- **Cobros y pagos a fondo**: la dimensión `agingBucket` y las medidas
  `daysToPayment` y `daysOverdue`, en `invoices` y `bills`. Sin ellos la
  consulta responde `403` con `code: plan_limit_exceeded` y
  `hint.features: ["collectionsAnalytics"]`.
- **Margen**: las medidas `cost`, `margin` y `marginRate` de `invoiceLines`.
  Sin él, `403` con `hint.features: ["marginAnalytics"]`.

Los nombres siguen en el vocabulario y en las descripciones de `describe`,
así que una aplicación puede anunciar lo que la empresa podría consultar si
ampliara su plan. Un administrador de la empresa lo amplía en FacturaDirecta;
volver a conectar la aplicación no lo soluciona. La contabilidad (`journal`)
no depende del plan en la API: la decide el scope `accounting:read`.

## Errores comunes

- `400 Bad Request`: una fuente, medida, dimensión o filtro fuera del
  vocabulario. El mensaje propone el nombre más parecido:
  «La medida 'totalbefore' no existe en 'invoices': ¿quieres decir
  'totalBeforeTaxes'?». Algunos sinónimos también tienen sugerencia:
  `taxId`, `nif` o `vat` proponen `fiscalId`. También `compare` y
  `breakdown` a la vez, más de 2 dimensiones, 10 medidas o 10 filtros,
  un periodo de más de 5 años o de más de 24 tramos, una fecha que no
  existe, un operador que el filtro no admite, `limit` fuera de 1 a 500 o
  `limitPerGroup` sin dos dimensiones.
- `403 Forbidden` (`token_scope_missing`): la credencial no tiene el
  scope de la fuente.
- `403 Forbidden` (`plan_limit_exceeded`): la consulta pide cifras de
  cobros a fondo o de margen que el plan de la empresa no incluye;
  `hint.features` dice cuál falta (ver [Planes](#planes)).
- `422 Unprocessable Entity` (`report_query_timeout`): la consulta ha
  tardado más de 30 segundos. Acota el periodo, las dimensiones o los
  filtros.

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| POST | `/{companyId}/query` | `analyticsQuery` | `accounting:read`, `bills:read`, `contacts:read`, `invoices:read` | Consulta analítica |

## Scopes

- **`accounting:read`** — Lectura de datos de contabilidad (diario).
- **`bills:read`** — Lectura de facturas de compra y tickets.
- **`contacts:read`** — Lectura de contactos.
- **`invoices:read`** — Lectura de facturas.
