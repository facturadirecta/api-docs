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

## Operaciones

- [Lista de productos](#lista-de-productos)
- [Crear producto](#crear-producto)
- [Obtener un producto](#obtener-un-producto)
- [Actualizar producto](#actualizar-producto)
- [Borrar producto](#borrar-producto)

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
  con `409 Conflict`. Para "retirar" un producto que ya se ha usado,
  archívalo en la interfaz o quita ambas facetas (`sales` y `purchases`).

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/products/pro_3c6b2e91-4d7a-4f1b-9e8c-2a5d7f0b1e4c"
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

## Errores comunes

- `400 ValidationError` — falta `name`, `currency` o alguno de los
  campos obligatorios de las facetas (`account`, `tax`).
- `400 ValidationError` — `tax` de la faceta `sales` con IDs del
  catálogo de compras (o viceversa). Ver
  [Impuestos](../guides/taxes.md).
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
| POST | `/{companyId}/products` | `createProduct` | `products:write` | Crear producto |
| PUT | `/{companyId}/products/{id}` | `updateProduct` | `products:write` | Actualizar producto |
| DELETE | `/{companyId}/products/{id}` | `deleteProduct` | `products:write` | Borrar producto |

## Scopes

- **`products:read`** — Lectura de productos.
- **`products:write`** — Modificación de productos.
