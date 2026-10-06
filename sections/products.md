---
title: Productos
audience:
  - developers
status: draft
---

# Productos

Un producto es un **concepto facturable preconfigurado** que se reutiliza
en líneas de documentos (facturas, presupuestos, albaranes, facturas de
compra). Tiene nombre, precio por defecto, impuestos por defecto y
cuentas contables asociadas. Al añadir un producto a una línea, los
campos de la línea se preinicializan con los valores del producto.

> En los ejemplos de esta página:
>
> - Los **UUIDs** (`pro_…`) son ilustrativos. Cada empresa tiene los
>   suyos; sustitúyelos por los identificadores reales que devuelve la API.
> - Los **IDs de impuestos** (`S_IVA_21`, `P_IVA_21_BC`) y las **cuentas
>   contables** (`700000`, `600000`) son los del catálogo por defecto.
>   Recupera los reales con `GET /{companyId}/settings/taxes/{sales,purchases}`.
>   Ver [Impuestos](../guides/taxes.md).

## Facetas: venta, compra, ambas

Un producto tiene dos facetas independientes y opcionales:

- **`content.main.sales`** — habilita al producto en documentos de venta
  (facturas, presupuestos, albaranes, recurrentes).
- **`content.main.purchases`** — habilita al producto en documentos de
  compra (facturas de compra y tickets).

No son excluyentes: un producto puede tener una, otra, o ambas. Sin
ninguna faceta el producto no es seleccionable en líneas (caso raro
pero válido para borradores).

Cada faceta tiene su propio bloque con dos campos obligatorios y
varios opcionales:

| Campo | Tipo | Obligatorio | Significado |
|---|---|---|---|
| `tax` | `TaxId[]` | sí | IDs del catálogo de impuestos correspondiente (ventas o compras). |
| `account` | `AccountCode` | sí | Cuenta contable: 700* en ventas, 600* en gastos corrientes, grupo 2 en inmovilizado amortizable. |
| `price` | number | no | Precio unitario por defecto. |
| `discountRate` | number | no | Descuento porcentual por defecto en **base 1** (`0.10` = 10%). |
| `description` | string | no | Texto que aparece en la línea del documento al incorporar el producto. |
| `depreciable` | boolean | no (solo en `purchases`) | Marca el producto como **bien amortizable**: el gasto se trata como inmovilizado sujeto a amortización en vez de gasto corriente. |
| `provider` | string | no (solo en `purchases`) | ID del contacto proveedor habitual. Se usa como proveedor por defecto de las líneas de pedido que incorporan el producto, para agrupar después las órdenes de compra por proveedor. |

## Estructura

- `content.type` — siempre `"product"`.
- `content.uuid` — identificador inmutable, prefijo `pro_`.
- `content.main`:
  - `name` (obligatorio) — nombre del producto.
  - `currency` (obligatorio) — código ISO 4217 (`EUR`, `USD`...).
  - `title` — calculado automáticamente; no enviar al crear.
  - `sku` (opcional) — referencia del producto.
  - `externalId` (opcional) — ID externo del producto. **Debe ser único
    entre todos los productos** de la empresa.
  - `sales` — faceta de venta (ver tabla anterior).
  - `purchases` — faceta de compra (ver tabla anterior).
  - `stock` — control de existencias del producto (ver [Control de
    stock](#control-de-stock)).

## Control de stock

El control de existencias es **opt-in por producto** y se activa con el
sub-objeto `content.main.stock`:

| Campo | Tipo | Obligatorio | Significado |
|---|---|---|---|
| `enabled` | boolean | sí | Con `true`, el producto controla existencias: los documentos configurados mueven su stock y cada movimiento queda registrado. |
| `minimum` | number | no | Stock mínimo deseado. Cuando el stock proyectado cae por debajo, el producto se marca como bajo mínimo y entra en la reposición sugerida. |
| `replenishLot` | number | no | Cantidad propuesta al reponer. Por defecto se propone la necesaria para volver al mínimo. |

Las **magnitudes** de stock (físico, comprometido, previsto) no forman parte
del producto: se consultan con [Stock de un producto](#stock-de-un-producto).

`main.stock` guarda solo la configuración. Si haces un `PUT` de producto sin
incluir `main.stock`, se conserva la configuración que ya tuviera el producto.

### Disponibilidad por plan

El control de stock forma parte del **Módulo Inventario**: está incluido en el
plan **Diamante** y se puede contratar como módulo en **Bronce**, **Plata** y
**Oro**. En el plan **Gratis** no está disponible.

Las tres operaciones de stock responden `403 Forbidden` con el código
`plan_limit_exceeded` y el mensaje «El control de stock no está disponible en
tu plan» si el plan no lo incluye.

Además, los planes limitan cuántos productos pueden llevar control de stock a
la vez (500 en Bronce y Plata; sin límite en Oro y Diamante). Al superarlo, la
API responde `403` con el código `plan_limit_exceeded`.

## Enlace a la aplicación

Las respuestas que devuelven un producto incluyen `webUrl` junto al resto de
los datos. En los listados, cada elemento de `items` lleva su propio enlace.

`webUrl` abre la ficha del elemento en la aplicación web de FacturaDirecta.
Puedes mostrarlo como enlace en tu integración sin construir rutas internas.
El navegador pedirá iniciar sesión con un usuario que tenga acceso a la
empresa. Es un campo opcional y no es un endpoint de la API: trátalo como una
URL para el usuario, no como una URL a la que enviar `$ACCESS_TOKEN`.

## Operaciones

- [Lista de productos](#lista-de-productos)
- [Crear producto](#crear-producto)
- [Obtener un producto](#obtener-un-producto)
- [Actualizar producto](#actualizar-producto)
- [Borrar producto](#borrar-producto)
- [Stock de un producto](#stock-de-un-producto)
- [Movimientos de stock de un producto](#movimientos-de-stock-de-un-producto)
- [Ajuste manual de stock](#ajuste-manual-de-stock)

## Lista de productos

`GET /{companyId}/products` devuelve los productos de la empresa,
paginados.

**Parámetros de consulta específicos:**

- **`title`** — búsqueda por título (calculado a partir del nombre).
- **`sku`** — búsqueda por referencia.
- **`externalId`** — búsqueda por ID externo.
- **`isSales`** — `true`/`false` para filtrar productos que tienen la
  faceta de venta.
- **`isPurchases`** — `true`/`false` para filtrar productos que tienen
  la faceta de compra.
- **`isDepreciable`** — `true`/`false` para filtrar productos marcados
  como amortizables (`purchases.depreciable: true`).
- **`salesDescription`** — búsqueda por descripción de venta.
- **`purchasesDescription`** — búsqueda por descripción de compra.
- **`sortBy`** — campo de orden.

**Parámetros globales aceptados:**

Acepta además los parámetros estándar `offset`, `limit`,
`minCreationDate`, `maxCreationDate`, `minModificationDate`,
`maxModificationDate` y el header `accept-version`. Ver
[Paginación](../guides/pagination.md) y
[Autenticación](../guides/authentication.md).

**Notas:**

- Los productos no tienen `related` ni filtros por etiquetas: son un
  recurso "hoja" sin relaciones que expandir.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/products?isSales=true&limit=50"
```

## Crear producto

`POST /{companyId}/products` crea un producto.

**Parámetros del body:**

- `content.type` — siempre `"product"`.
- `content.main.name` — obligatorio.
- `content.main.currency` — obligatorio, ISO 4217.
- `content.main.sales` y/o `content.main.purchases` — al menos una de
  las dos facetas en uso típico. Cada faceta requiere `tax` y `account`.
- `tags` (opcional).

**Notas:**

- `title` se calcula automáticamente; no es necesario enviarlo.
- Si envías `externalId`, debe ser único entre los productos de la
  empresa. Si choca con uno existente, la API devuelve `409 Conflict`.
- La respuesta es el producto creado completo, en el mismo formato que
  [Obtener un producto](#obtener-un-producto).

**Parámetros globales aceptados:** `accept-version`.

###### Ejemplo de request JSON

Producto de solo venta (heredado del ejemplo `sales` del openapi):

```json
{
  "content": {
    "type": "product",
    "main": {
      "sku": "PV001",
      "name": "Producto 001",
      "currency": "EUR",
      "sales": {
        "price": 125,
        "description": "Descripción del producto 001 que aparecerá en los documentos cuando se seleccione",
        "tax": ["S_IVA_21"],
        "account": "700000"
      }
    }
  }
}
```

Producto de venta y compra (heredado del ejemplo `salesAndPurchases`):

```json
{
  "content": {
    "type": "product",
    "main": {
      "sku": "PCV009",
      "name": "Producto 009",
      "currency": "EUR",
      "sales": {
        "price": 125,
        "description": "Descripción del producto en los documentos de venta",
        "tax": ["S_IVA_21"],
        "account": "700000"
      },
      "purchases": {
        "price": 75,
        "description": "Descripción del producto en los documentos de compra",
        "tax": ["P_IVA_21_BC"],
        "account": "600000"
      }
    }
  }
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '@product.json' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/products"
```

## Obtener un producto

`GET /{companyId}/products/{id}` devuelve un producto por ID.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/products/pro_3c6b2e91-4d7a-4f1b-9e8c-2a5d7f0b1e4c"
```

## Actualizar producto

`PUT /{companyId}/products/{id}` sustituye **el contenido completo** del
producto. No es un PATCH.

**Notas:**

- Cambiar el precio de un producto **no modifica documentos ya emitidos**
  que usaron ese producto. Solo afecta a las nuevas líneas creadas con
  el producto a partir de ahora.
- Para retirar una faceta, omite el sub-objeto correspondiente en el
  PUT (no envíes `sales: null`; simplemente no lo incluyas).

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '@product.json' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/products/pro_3c6b2e91-4d7a-4f1b-9e8c-2a5d7f0b1e4c"
```

## Borrar producto

`DELETE /{companyId}/products/{id}` elimina un producto.

**Restricciones:**

- Si el producto está referenciado por alguna línea de documento (a
  través del campo `document` de la línea), la API rechaza el borrado
  con `409 Conflict`. Para retirar un producto que ya se ha usado, quita
  ambas facetas (`sales` y `purchases`) para que deje de ser seleccionable.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/products/pro_3c6b2e91-4d7a-4f1b-9e8c-2a5d7f0b1e4c"
```

## Stock de un producto

`GET /{companyId}/products/{id}/stock` devuelve las magnitudes de stock del
producto, **sumadas de todos los almacenes**.

Requiere que el producto tenga el control de stock activado
(`main.stock.enabled: true`). Si no lo tiene, la respuesta es `400` con el
mensaje «El producto no tiene el control de stock activado».

**Campos de la respuesta** (todos dentro de `content`):

| Campo | Significado |
|---|---|
| `physical` | Existencias físicas: la suma del libro de movimientos. |
| `committed` | Comprometido en pedidos de cliente activos. |
| `incoming` | Previsto en órdenes de compra pendientes de recibir. |
| `available` | Disponible: `physical − committed`. |
| `projected` | Proyectado: `available + incoming`. |

`committed` e `incoming` salen de las líneas de
[pedidos](./client-orders.md) y [órdenes de compra](./purchase-orders.md); ver
[Flujo de pedidos y órdenes de compra](../guides/orders-flow.md).

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/products/pro_3c6b2e91-4d7a-4f1b-9e8c-2a5d7f0b1e4c/stock"
```

Respuesta:

```json
{
  "content": {
    "physical": 5,
    "committed": 0,
    "incoming": 0,
    "available": 5,
    "projected": 5
  }
}
```

## Movimientos de stock de un producto

`GET /{companyId}/products/{id}/stockMovements` devuelve el libro de
movimientos del producto, **del más reciente al más antiguo**, con el saldo
tras cada movimiento.

Requiere el control de stock activado en el producto.

**Campos de cada movimiento:**

| Campo | Significado |
|---|---|
| `id` | Identificador del movimiento. |
| `date` | Fecha efectiva del movimiento. |
| `quantity` | Cantidad con signo: positiva en entradas, negativa en salidas. |
| `type` | `initial`, `reception`, `delivery`, `adjustment`, `count` o `transfer`. |
| `warehouse` | Identificador del almacén donde se registró. |
| `balanceAfter` | Saldo del producto **en ese almacén** tras el movimiento. |
| `transferGroup` | Identificador compartido por las dos patas de una transferencia entre almacenes. |
| `reason` | Motivo, solo en ajustes manuales (ver [Ajuste manual de stock](#ajuste-manual-de-stock)). |
| `sourceDocument` | ID del documento que generó el movimiento, si lo hay. |
| `note` | Nota libre del movimiento. |
| `creationDate` | Fecha y hora de registro en el sistema. |

Como `balanceAfter` es el saldo **del almacén** y no el del producto entero, en
empresas con varios almacenes no coincide con el `physical` de
[Stock de un producto](#stock-de-un-producto).

**Parámetros globales aceptados:**

Acepta `offset`, `limit` y el header `accept-version`. La respuesta trae
`pagination` e `items`. Ver [Paginación](../guides/pagination.md).

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/products/pro_3c6b2e91-4d7a-4f1b-9e8c-2a5d7f0b1e4c/stockMovements?limit=50"
```

Respuesta:

```json
{
  "pagination": { "offset": 0, "limit": 50, "total": 1 },
  "items": [
    {
      "id": "stm_4d9a7c21-3b5e-4f80-9a1c-6d2e8f0b3a5c",
      "date": "2026-09-12",
      "quantity": 5,
      "type": "adjustment",
      "warehouse": "war_main",
      "reason": "correccion",
      "note": "alta inicial por API",
      "creationDate": "2026-09-12T09:14:22.000Z",
      "balanceAfter": 5
    }
  ]
}
```

## Ajuste manual de stock

`POST /{companyId}/products/{id}/stockAdjustments` registra un ajuste manual
del stock físico con un motivo tipado. El ajuste queda como un movimiento más
del libro.

Requiere el control de stock activado en el producto.

**Parámetros del body:**

- `quantity` (obligatorio) — cantidad con signo: positiva para dar entrada,
  negativa para dar salida. **No puede ser cero.**
- `reason` (obligatorio) — motivo del ajuste: `merma`, `rotura`, `robo`,
  `caducidad`, `consumo_propio`, `muestra_regalo` o `correccion`.
- `note` (opcional) — nota libre.

**Notas:**

- El ajuste se registra siempre en el **almacén por defecto**: este endpoint no
  acepta almacén. Para ajustar otro almacén, usa la interfaz.
- Si la empresa tiene configurado bloquear los negativos y el ajuste dejaría
  ese almacén por debajo de cero, la API responde `400` y **no** registra el
  movimiento.
- La respuesta devuelve el `id` del movimiento creado, la `quantity` aplicada y
  el `physical` resultante del producto (suma de todos los almacenes).

**Parámetros globales aceptados:** `accept-version`.

###### Ejemplo de request JSON

```json
{
  "quantity": 5,
  "reason": "correccion",
  "note": "alta inicial por API"
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"quantity":5,"reason":"correccion","note":"alta inicial por API"}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/products/pro_3c6b2e91-4d7a-4f1b-9e8c-2a5d7f0b1e4c/stockAdjustments"
```

Respuesta:

```json
{
  "content": {
    "id": "stm_4d9a7c21-3b5e-4f80-9a1c-6d2e8f0b3a5c",
    "quantity": 5,
    "physical": 5
  }
}
```

## Recomendaciones

- **Configura ambas facetas si el producto se compra y se vende**
  (típico en distribución / retail): así la línea se preinicializa con
  los datos correctos según el documento donde se incorpora.
- **Usa `externalId`** si tu producto ya existe en otro sistema (ERP,
  Shopify, etc.) y necesitas mapearlo de forma idempotente. La unicidad
  por empresa te permite usarlo como llave para upserts.
- **Para activos amortizables** (equipos, mobiliario), pon
  `purchases.depreciable: true` y elige una cuenta del grupo 2
  (inmovilizado) en `purchases.account`. La interfaz aplicará el
  tratamiento contable correspondiente al crear el gasto.
- **No envíes `title`**: lo calcula el servidor a partir del `name`.
- **Activa el control de stock solo donde aporte.** Cada producto con
  `stock.enabled: true` consume uno de los huecos que permite el plan, y el
  libro de movimientos crece con cada documento que lo mueve.
- **Para inventarios iniciales**, registra la existencia de partida con un
  [ajuste manual](#ajuste-manual-de-stock) de motivo `correccion` en lugar de
  crear documentos ficticios.
- **Antes de asignar un proveedor a líneas de pedido**, comprueba que el
  producto tiene la faceta `purchases`: sin ella, la línea se rechaza.

## Versiones

`GET /{companyId}/products/{id}/versions` lista las versiones de el producto: su alta, cada
modificación, su eliminación y su recuperación, con quién y cuándo, aunque se haya eliminado.
`GET /{companyId}/products/{id}/versions/{versionId}` devuelve la copia de una versión, en el
mismo formato que `GET /{companyId}/products/{id}`, y los cambios respecto a la versión anterior.
Sirven para ver qué se cambió y recuperar datos: ver
[Versiones de documentos](../guides/document-versions.md).

## Errores comunes

- `400 ValidationError` — falta `name`, `currency` o alguno de los
  campos obligatorios de las facetas (`account`, `tax`).
- `400 ValidationError` — `tax` de la faceta `sales` con IDs del
  catálogo de compras (o viceversa). Ver
  [Impuestos](../guides/taxes.md).
- `400 ValidationError` — se intenta retirar la faceta `purchases` de un
  producto que participa en pedidos pendientes u órdenes de compra abiertas.
- `400` — ajuste de stock con `quantity` a cero, o ajuste que dejaría el
  almacén en negativo cuando la empresa bloquea los negativos.
- `400` — operación de stock sobre un producto que no tiene el control de
  stock activado.
- `403` con código `plan_limit_exceeded` — se ha alcanzado el máximo de
  productos del plan, o el máximo de productos con control de stock.
- `403` — el plan no incluye el control de stock (ver
  [Disponibilidad por plan](#disponibilidad-por-plan)).
- `409 Conflict` — borrado de un producto referenciado por líneas de
  documentos existentes.
- `409 Conflict` — `externalId` duplicado en la empresa.

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
| GET | `/{companyId}/products` | `getProducts` | `products:read` | Lista de productos |
| GET | `/{companyId}/products/{id}` | `getProduct` | `products:read` | Obtener un producto |
| GET | `/{companyId}/products/{id}/stock` | `getProductStock` | `products:read` | Stock de un producto |
| GET | `/{companyId}/products/{id}/stockMovements` | `listProductStockMovements` | `products:read` | Movimientos de stock de un producto |
| GET | `/{companyId}/products/{id}/versions` | `getProductVersions` | `products:read` | Versiones de el producto |
| GET | `/{companyId}/products/{id}/versions/{versionId}` | `getProductVersion` | `products:read` | Versión de el producto |
| POST | `/{companyId}/products` | `createProduct` | `products:write` | Crear producto |
| POST | `/{companyId}/products/{id}/stockAdjustments` | `adjustProductStock` | `products:write` | Ajuste manual de stock |
| PUT | `/{companyId}/products/{id}` | `updateProduct` | `products:write` | Actualizar producto |
| DELETE | `/{companyId}/products/{id}` | `deleteProduct` | `products:write` | Borrar producto |

## Scopes

- **`products:read`** — Lectura de productos.
- **`products:write`** — Modificación de productos.
