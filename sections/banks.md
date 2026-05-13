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
se configuran las credenciales de Afterbanks/ArcoPay, los datos de
SEPA y demás detalles operativos. La API solo expone los datos
necesarios para integraciones (registrar pagos contra una cuenta,
mostrar listado al usuario).

## Estructura

Cada item del listado tiene la forma `{ uuid, content, creationDate,
modificationDate }`. El contenido principal está en `content.main`:

| Campo | Tipo | Significado |
|---|---|---|
| `name` | string | Nombre de la cuenta. |
| `title` | string | Título descriptivo (opcional). |
| `subtype` | enum | `iban`, `card` u `other`. Determina qué otros campos aplican. |
| `currency` | string | Moneda (ISO 4217). |
| `account` | string | Cuenta contable del banco en el plan contable. |
| `iban4` | string | Últimos 4 dígitos del IBAN, prefijados por `****`. Solo si `subtype="iban"`. |
| `iban` | string | IBAN completo. Solo si `subtype="iban"` **Y** las credenciales tienen el scope `banks:readIban`. |
| `bic` | string | BIC/SWIFT (si está informado). |
| `bankCode` | string | Código de banco usado en exportaciones a contabilidad externa (A3). |

> **No hay tags** sobre bancos. A diferencia del resto de documentos,
> el item devuelto no incluye el campo `tags`.

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

## Operaciones

- [Lista de bancos](#lista-de-bancos)

## Lista de bancos

`GET /{companyId}/banks` devuelve los bancos activos de la empresa.

**Parámetros de consulta específicos:**

- **`currency`** — filtrar por moneda (ISO 4217, exacto).
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

- El listado **excluye los bancos archivados**: solo aparecen las
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

## Recomendaciones

- **Cachea el listado**. Las cuentas cambian con muy baja frecuencia;
  no hace falta consultar el endpoint en cada operación. Refresca el
  cache cuando una creación de cobro/pago falle con `400` por `bank`
  desconocido (alguien puede haber archivado o añadido cuentas desde
  la interfaz).
- **Para mostrar al usuario**, usa `name` (o `title` si está
  informado) e `iban4`. No pidas `banks:readIban` solo para mostrar
  la cuenta: con `iban4` ya tienes los últimos 4 dígitos.
- **Para registrar pagos** (`POST /bills/{id}/payments` o
  `POST /payrolls/{id}/payments`), basta con el `uuid` del banco.

## Errores comunes

- **No hay `404` específico**: este endpoint solo devuelve lista. Si
  no hay bancos, recibirás `items: []`.
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

## Scopes

- **`banks:read`** — Lectura de definiciones de bancos y cuentas de tesorería.
