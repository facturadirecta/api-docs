---
title: Bancos y cuentas de tesorería
audience:
  - developers
status: draft
---

# Bancos y cuentas de tesorería

Un **banco** (`bank`) en FacturaDirecta es una cuenta de tesorería de la
empresa: una cuenta corriente con IBAN, una cuenta de tarjeta o una
caja en efectivo. Los bancos son las contrapartidas naturales de los
cobros y pagos, y son lo que se concilia contra los
[extractos bancarios](./statements.md).

> En los ejemplos, `ban_…` son ilustrativos. El ID real se devuelve al
> consultar la API.

## Los bancos no se crean por API

Este recurso es **solo lectura**. La API pública no expone crear,
modificar ni borrar bancos: ese flujo vive solo en la interfaz, donde
se configura la conexión automática con el banco, los datos de SEPA y demás
detalles operativos. La API expone los datos
necesarios para integraciones (registrar pagos contra una cuenta,
mostrar listado al usuario) y, con el parámetro `related`, el saldo
contable, el saldo según la entidad, el estado del enlace y los
movimientos pendientes de conciliar: lo mismo que muestra la pantalla
de bancos de la aplicación.

## Estructura

Cada item del listado tiene la forma `{ uuid, content, creationDate,
modificationDate }`. El contenido principal está en `content.main`:

| Campo | Tipo | Significado |
|---|---|---|
| `name` | string | Nombre de la cuenta. |
| `title` | string | Título descriptivo (opcional). |
| `notes` | string | Notas internas de la cuenta, si están informadas en la interfaz. |
| `subtype` | enum | `iban`, `card`, `stripe` u `other`. Determina qué otros campos aplican. Un banco `stripe` es una cuenta de Stripe: no tiene IBAN y sus movimientos no traen saldo de la entidad. |
| `currency` | string | Moneda (ISO 4217). |
| `account` | string | Cuenta contable del banco en el plan contable. |
| `iban4` | string | Últimos 4 dígitos del IBAN, prefijados por `****`. Solo si `subtype="iban"`. |
| `iban` | string | IBAN completo. Solo si `subtype="iban"` **Y** las credenciales tienen el scope `banks:readIban`. |
| `bic` | string | BIC/SWIFT (si está informado). |
| `bankCode` | string | Código de banco usado en exportaciones a contabilidad externa (A3). |

> **No hay tags** sobre bancos. A diferencia del resto de documentos,
> el item devuelto no incluye el campo `tags`.

## Saldos y estado del enlace: el parámetro `related`

El listado y el detalle aceptan `related`, repetible, con estos valores.
Cada uno añade una clave bajo `related` en el item; sin el parámetro, la
respuesta no lleva `related`.

| Valor | Clave | Qué devuelve |
|---|---|---|
| `balance` | `related.balance` | Saldo contable del banco. |
| `linkedBalance` | `related.linkedBalance` | Saldo según la entidad enlazada, o `null`. |
| `link` | `related.link` | Estado de la conexión automática con el banco. |
| `statements` | `related.statements` | Movimientos pendientes de conciliar y fecha del último. |

Todos los importes de `related` son positivos cuando hay dinero en el
banco, igual que `amount` en los [extractos](./statements.md).

### Los dos saldos

Un banco tiene dos saldos distintos, y conviene no confundirlos:

- **`related.balance`, el saldo contable**: la suma de los apuntes de la
  cuenta contable del banco (`account`) cuyo tercero es el banco. Es el
  saldo de FacturaDirecta: el de la pantalla de bancos, el del panel de
  inicio y el de [sumas y saldos](./reports.md). Está en todos los
  planes. Para sincronizarlo en un ERP o cuadrar la contabilidad, este
  es el dato.
- **`related.linkedBalance`, el saldo según la entidad**: el que el banco
  informó en el último movimiento sincronizado o importado. Es un dato
  informativo que depende de hasta dónde ha sincronizado el proveedor:
  cambia con cada sincronización (cada 12 horas, o cuando un usuario la
  pide desde la aplicación) y puede ir por detrás del banco real. Por
  eso lleva la fecha y el id del movimiento del que sale y la fecha de la
  última sincronización correcta: con ellos tu integración decide si le
  sirve.

La diferencia entre los dos suele ser lo que falta por conciliar:
`related.statements.pending` dice cuántos movimientos son.

`related.balance`:

| Campo | Tipo | Significado |
|---|---|---|
| `balance` | number | Saldo en la moneda de la empresa. Suma todos los apuntes del banco, en cualquier moneda, por su importe convertido. |
| `currency` | string | Moneda del banco (ISO 4217). |
| `currencyBalance` | number | Saldo en la moneda del banco. Suma solo los apuntes en esa moneda, por su importe en divisa. En un banco en la moneda de la empresa coincide con `balance`. |
| `latestEntryDate` | date | Fecha del último apunte que mueve el saldo, o `null` si no hay apuntes. |
| `date` | date | Fecha de corte. Solo cuando se pasa `balanceDate`. |

Sin `balanceDate`, el saldo incluye **todos los apuntes, también los de
fecha futura**: es el mismo número que enseña la pantalla de bancos. Con
`balanceDate=YYYY-MM-DD` es el saldo al cierre de ese día (apuntes con
fecha hasta ese día inclusive), y `latestEntryDate` es la del último
apunte dentro del corte. `balanceDate` solo tiene efecto con
`related=balance`.

`related.linkedBalance` (o `null` si no hay saldo de la entidad: banco
sin movimientos, movimientos importados sin saldo, o banco `stripe`):

| Campo | Tipo | Significado |
|---|---|---|
| `balance` | number | Saldo disponible que informó la entidad tras el movimiento. |
| `bookedBalance` | number | Saldo contable que informó la entidad, cuando difiere del disponible. `null` si no lo informa. |
| `currency` | string | Moneda del saldo (ISO 4217). |
| `date` | date | Fecha del movimiento del que sale el saldo. |
| `statement` | string | Id del movimiento del que sale el saldo (`GET /statements/{id}`). |
| `syncedAt` | date-time | Última sincronización correcta con el proveedor. `null` si el banco no sincroniza (movimientos importados por fichero). |

El saldo es el del último movimiento con fecha hasta hoy, esté o no
conciliado o ignorado: es el último dato que dio la entidad.

`related.link`:

| Campo | Tipo | Significado |
|---|---|---|
| `status` | enum | `manual` (sin conexión; los movimientos se importan a mano), `linked` (conectado con el banco, o cuenta de Stripe conectada), `paused` (conectado, pero el plan no incluye más cuentas conectadas y la sincronización está pausada) o `unlinked` (estuvo conectado y ya no). |
| `bankName` | string | Nombre de la entidad según la conexión. `null` en un banco manual. |
| `accessValidUntil` | date | Hasta cuándo autorizó el usuario el acceso a los datos (PSD2). `null` si el proveedor no lo informa. |
| `syncEnabled` | boolean | Si el enlace sincroniza movimientos automáticamente. |
| `actionRequired` | boolean | Si la sincronización está detenida porque el usuario debe hacer algo en la aplicación (volver a autorizar el acceso, confirmar un código…). |
| `message` | string | Motivo por el que está detenida, tal como lo ve el usuario. `null` si no lo está. |
| `lastSyncAt` | date-time | Último intento de sincronización. `null` si nunca ha sincronizado. |
| `lastSyncOk` | boolean | Si el último intento acabó bien. `null` si nunca ha sincronizado. |

`status` solo no dice si el banco sincroniza: un banco de Stripe es
`linked` en cuanto tiene la cuenta conectada para cobrar, aunque no
sincronice movimientos. Mira `syncEnabled`. `related.link` nunca incluye
credenciales ni referencias de cuenta del proveedor.

`related.statements`:

| Campo | Tipo | Significado |
|---|---|---|
| `pending` | integer | Movimientos pendientes de conciliar. Quedan fuera los ignorados y los borrados; un movimiento cuyo asiento se ha borrado vuelve a contar como pendiente. |
| `latestDate` | date | Fecha del último movimiento del banco con fecha hasta hoy. `null` si no tiene movimientos. |

Son solo contadores y fechas: el listado de movimientos está en
[`GET /{companyId}/statements?bank={id}`](./statements.md).

## Acceso al IBAN: dos scopes

Los IBAN son datos sensibles. La API segrega el acceso en dos niveles:

- **`banks:read`** — siempre presente para listar bancos. Devuelve
  todos los campos **excepto `iban`**. El `iban4` (últimos 4 dígitos
  enmascarados) sí se devuelve.
- **`banks:readIban`** — scope adicional para ver el IBAN completo.
  Sin él, `iban` se omite del objeto.

Para integraciones que solo necesitan identificar la cuenta (UI de
selección, conciliación contra banco), basta con `banks:read`. Pide
`banks:readIban` solo si tu integración realmente necesita el IBAN
completo (por ejemplo, generación de remesas SEPA externas).

> Este patrón de doble scope se aplica también en los
> [métodos de pago](./payment-methods.md) (allí se enmascara el IBAN
> del deudor del SEPA Direct Debit por la misma razón).

## Enlace a la aplicación

Las respuestas que devuelven un banco incluyen `webUrl` junto al resto de
los datos. En los listados, cada elemento de `items` lleva su propio enlace.

`webUrl` abre la ficha del elemento en la aplicación web de FacturaDirecta.
Puedes mostrarlo como enlace en tu integración sin construir rutas internas.
El navegador pedirá iniciar sesión con un usuario que tenga acceso a la
empresa. Es un campo opcional y no es un endpoint de la API: trátalo como una
URL para el usuario, no como una URL a la que enviar `$ACCESS_TOKEN`.

## Operaciones

- [Lista de bancos](#lista-de-bancos)
- [Obtener un banco](#obtener-un-banco)

## Lista de bancos

`GET /{companyId}/banks` devuelve los bancos activos de la empresa.

**Parámetros de consulta específicos:**

- **`currency`** — filtrar por moneda (ISO 4217, exacto).
- **`related`** — datos adicionales de cada banco: `balance`,
  `linkedBalance`, `link`, `statements` (repetible). Ver
  [Saldos y estado del enlace](#saldos-y-estado-del-enlace-el-parámetro-related).
- **`balanceDate`** — fecha de corte del saldo contable (`YYYY-MM-DD`).
  Solo con `related=balance`.
- **`sortBy`** — campo de orden. Valores aceptados: `title`,
  `currency`, `creationDate`, `modificationDate` (cada uno con su
  variante descendente `-title`...).

**Parámetros globales aceptados:**

Acepta además los parámetros estándar `offset`, `limit`,
`minCreationDate`, `maxCreationDate`, `minModificationDate`,
`maxModificationDate` y el header `accept-version`. Ver
[Paginación](../guides/pagination.md) y
[Autenticación](../guides/authentication.md).

**Notas:**

- El listado **excluye los bancos borrados**: solo aparecen las
  cuentas activas de la empresa.
- El orden por defecto es por número total de comentarios y luego
  por fecha de creación descendente. Si tu integración depende del
  orden, indícalo explícitamente con `sortBy`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/banks?currency=EUR&sortBy=title"
```

Respuesta típica (sin scope `banks:readIban`):

```json
{
  "pagination": { "offset": 0, "limit": 100, "total": 2 },
  "items": [
    {
      "uuid": "ban_5e7d8a31-9c4b-4f6e-a1d3-2b5c7e9f1a4d",
      "content": {
        "type": "bank",
        "uuid": "ban_5e7d8a31-9c4b-4f6e-a1d3-2b5c7e9f1a4d",
        "main": {
          "name": "Cuenta principal",
          "subtype": "iban",
          "currency": "EUR",
          "notes": "Cuenta para cobros principales",
          "account": "572000",
          "iban4": "****1234",
          "bic": "BSCHESMMXXX"
        }
      },
      "creationDate": "2024-01-15T08:00:00.000Z",
      "modificationDate": "2024-01-15T08:00:00.000Z"
    }
  ]
}
```

Con `related` cada item añade los bloques pedidos:

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/banks?related=balance&related=statements"
```

## Obtener un banco

`GET /{companyId}/banks/{id}` devuelve un banco activo, con la misma
forma que cada item del listado (`{ uuid, content, creationDate,
modificationDate, webUrl }`) y los mismos scopes. Acepta `related` y
`balanceDate`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/banks/ban_5e7d8a31-9c4b-4f6e-a1d3-2b5c7e9f1a4d?related=balance&related=linkedBalance&related=link&related=statements"
```

Respuesta típica de un banco enlazado por PSD2:

```json
{
  "uuid": "ban_5e7d8a31-9c4b-4f6e-a1d3-2b5c7e9f1a4d",
  "content": {
    "type": "bank",
    "uuid": "ban_5e7d8a31-9c4b-4f6e-a1d3-2b5c7e9f1a4d",
    "main": {
      "name": "Cuenta principal",
      "subtype": "iban",
      "currency": "EUR",
      "account": "572000",
      "iban4": "****1234",
      "bic": "BSCHESMMXXX"
    }
  },
  "creationDate": "2024-01-15T08:00:00.000Z",
  "modificationDate": "2026-09-02T10:21:00.000Z",
  "webUrl": "https://3.facturadirecta.com/company/com_…/banks/ban_5e7d8a31-9c4b-4f6e-a1d3-2b5c7e9f1a4d",
  "related": {
    "balance": {
      "balance": 12345.67,
      "currency": "EUR",
      "currencyBalance": 12345.67,
      "latestEntryDate": "2026-10-15"
    },
    "linkedBalance": {
      "balance": 12400.12,
      "bookedBalance": null,
      "currency": "EUR",
      "date": "2026-10-15",
      "statement": "sta_8c1f2b7e-3d4a-4e5f-9a6b-7c8d9e0f1a2b",
      "syncedAt": "2026-10-16T06:12:41.000Z"
    },
    "link": {
      "status": "linked",
      "bankName": "Banco Santander",
      "accessValidUntil": "2026-12-01",
      "syncEnabled": true,
      "actionRequired": false,
      "message": null,
      "lastSyncAt": "2026-10-16T06:12:41.000Z",
      "lastSyncOk": true
    },
    "statements": { "pending": 3, "latestDate": "2026-10-15" }
  }
}
```

Los 54,45 € de diferencia entre los dos saldos son los tres movimientos
pendientes de conciliar: cuando se concilien, el saldo contable coincidirá
con el de la entidad.

Para el saldo contable **a una fecha** (por ejemplo, al cierre de un
mes):

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/banks/ban_5e7d8a31-9c4b-4f6e-a1d3-2b5c7e9f1a4d?related=balance&balanceDate=2026-09-30"
```

`related.balance` lleva entonces `"date": "2026-09-30"` y solo suma los
apuntes con fecha hasta ese día. El mismo saldo, con el detalle de cada
cuenta, lo da [sumas y saldos](./reports.md) con `thirdParty` y `account`,
pero ese informe exige el módulo de Contabilidad y el scope
`accounting:read`; `related=balance` está en todos los planes.

## Recomendaciones

- **Para el saldo de un banco** usa `related=balance` en el detalle o en
  el listado. No sumes los apuntes de `GET /journal` ni construyas el
  saldo a partir de los extractos: es el mismo número que ve el usuario
  en la pantalla de bancos, calculado con el mismo criterio.
- **No uses `linkedBalance` como saldo contable.** Es el último dato que
  dio la entidad, con fecha; sirve para mostrar «saldo en el banco» o para
  detectar que la sincronización va atrasada (`syncedAt`), no para cuadrar
  la contabilidad.
- **Cachea el listado**. Las cuentas cambian con muy baja frecuencia;
  no hace falta consultar el endpoint en cada operación. Refresca el
  cache cuando una creación de cobro/pago falle con `400` por `bank`
  desconocido (alguien puede haber borrado o añadido cuentas desde
  la interfaz).
- **Para mostrar al usuario**, usa `name` (o `title` si está
  informado) e `iban4`. No pidas `banks:readIban` solo para mostrar
  la cuenta: con `iban4` ya tienes los últimos 4 dígitos.
- **Para registrar pagos** (`POST /bills/{id}/payments` o
  `POST /payrolls/{id}/payments`), basta con el `uuid` del banco.

## Versiones

`GET /{companyId}/banks/{id}/versions` lista las versiones de el banco: su alta, cada
modificación, su eliminación y su recuperación, con quién y cuándo, aunque se haya eliminado.
`GET /{companyId}/banks/{id}/versions/{versionId}` devuelve la copia de una versión, en el
mismo formato que el listado `GET /{companyId}/banks`, y los cambios respecto a la versión anterior.
Sirven para ver qué se cambió y recuperar datos: ver
[Versiones de documentos](../guides/document-versions.md).

## Errores comunes

- El listado no devuelve `404`: si no hay bancos, recibirás
  `items: []`.
- `404 Not Found` en `GET /banks/{id}` — el banco no existe o está
  borrado.
- `400 Bad Request` — un valor de `related` que no existe, o un
  `balanceDate` que no es una fecha `YYYY-MM-DD`.
- `403 Forbidden` (`code: "forbidden"`) — falta el scope
  `banks:read`.

Ver [Errores y validaciones](../guides/errors.md) para el formato
general.

## Referencia exhaustiva

Esta página cubre los matices funcionales. Para el contrato completo
del schema (incluyendo los `oneOf` por `subtype`), consulta el
[Swagger UI](https://www.facturadirecta.com/api) o el
[openapi crudo](https://app.facturadirecta.com/openapi.json).

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| GET | `/{companyId}/banks` | `getBanks` | `banks:read` | Lista de bancos |
| GET | `/{companyId}/banks/{id}` | `getBank` | `banks:read` | Obtener un banco |
| GET | `/{companyId}/banks/{id}/versions` | `getBankVersions` | `banks:read` | Versiones de el banco |
| GET | `/{companyId}/banks/{id}/versions/{versionId}` | `getBankVersion` | `banks:read` | Versión de el banco |

## Scopes

- **`banks:read`** — Lectura de bancos y cuentas de tesorería, sus saldos y el estado de su enlace.
