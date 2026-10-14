---
title: Búsqueda
audience:
  - developers
status: draft
---

# Búsqueda

La **búsqueda global** localiza en una sola llamada cualquier elemento de la
empresa a partir de un texto: un cliente por su nombre o su NIF, una factura
por su número, un producto por su referencia o un gasto por el texto de sus
líneas. Es la misma búsqueda que ofrece la paleta de comandos de la
aplicación (`Cmd + K`), con el mismo criterio de coincidencia que la búsqueda
rápida de cada listado.

Sirve para **encontrar** y después leer el elemento con el `GET` de su recurso,
o para enlazarlo con su `webUrl`. Para listar, paginar o filtrar por fechas,
estados o importes están los listados de cada recurso, que admiten el mismo
texto en su parámetro `search`.

## Qué se busca

Entran los tipos que tienen búsqueda rápida en su listado de la aplicación, en
este orden, que es también el orden de los grupos de la respuesta:

| `type` | `resource` | Dónde se busca |
|---|---|---|
| `contact` | `contacts` | Nombre y apellidos, nombre comercial, NIF, IVA intracomunitario, email, teléfono, personas de contacto y direcciones. |
| `invoice` | `invoices` | Número, título, cliente (nombre, apellidos, NIF), dirección y texto de las líneas. |
| `estimate` | `estimates` | Como las facturas. |
| `deliveryNote` | `deliveryNotes` | Como las facturas. |
| `clientOrder` | `clientOrders` | Como las facturas. |
| `purchaseOrder` | `purchaseOrders` | Como las facturas. |
| `bill` | `bills` | Referencia, título, proveedor, notas y texto de las líneas. Incluye los tickets (`displayType: ticket`). |
| `payroll` | `payrolls` | Empleado, título, notas y texto de las líneas. |
| `product` | `products` | Título, referencia (SKU) y descripciones de venta y de compra. |
| `automation` | `recurring` | Título y texto de la plantilla de las facturas recurrentes (`GET /{companyId}/recurring/invoices`). |

Los documentos archivados no se devuelven. Los pagos, los movimientos
bancarios, los asientos y la bandeja de entrada no entran en la búsqueda: se
consultan con sus propios recursos.

## Cómo coincide el texto

- Se buscan **todas las palabras** del texto, en cualquier orden y posición,
  sin distinguir mayúsculas ni acentos: `acme consultoria` encuentra la
  factura de *Acme Soluciones SL* con una línea «Consultoría trimestral».
- El **número del documento** se compara normalizado: sin ceros a la
  izquierda y con guiones y barras equivalentes a espacios. `F-2026-12`,
  `F 2026 12` y `F-2026-0012` encuentran la misma factura. En las facturas de
  compra y tickets se compara la referencia del proveedor.
- Dentro de cada tipo, las coincidencias de número van primero; el resto, de
  más reciente a más antiguo.

## Permisos

La búsqueda no tiene un scope propio: **cada tipo entra con el scope de
lectura de su recurso** (`contacts:read`, `invoices:read`, `estimates:read`,
`deliveryNotes:read`, `clientOrders:read`, `purchaseOrders:read`,
`bills:read`, `payrolls:read`, `products:read`, `recurring:read`) y con las
mismas reglas que su listado:

- Una **API key** busca en los recursos cuyos scopes tiene.
- Con **OAuth** manda además el rol del usuario en la empresa: un usuario del
  rol *Ventas* no recibe facturas de compra ni proveedores aunque el token
  lleve esos scopes. Con permisos V3 (roles personalizados, aplicaciones
  conectadas), en contactos solo las facetas permitidas (clientes,
  proveedores, empleados).
- Las facturas recurrentes exigen `recurring:read` e `invoices:read`, como su
  listado.
- Si el token no puede leer ningún tipo, la respuesta es `403` con el código
  `token_scope_missing`.

## Lista de resultados

`GET /{companyId}/search?q=acme`

**Parámetros de consulta:**

- `q` — texto a buscar, de 2 a 200 caracteres. Obligatorio.
- `types` — recursos a los que acotar la búsqueda (`invoices`, `contacts`…),
  varios separados por comas (`types=invoices,contacts`) o repitiendo el
  parámetro (`types=invoices&types=contacts`). Sin indicar, todos los que el
  token puede leer. Un valor desconocido devuelve `400`.
- `limitPerType` — resultados por tipo, de 1 a 20 (por defecto 5).

No hay paginación: la búsqueda devuelve los primeros resultados de cada tipo y
avisa con `hasMore` de que hay más. Para verlos todos, el listado del recurso
con el mismo texto en `search` (por ejemplo
`GET /{companyId}/invoices?search=acme`), que sí pagina y admite el resto de
filtros.

**Respuesta:**

```json
{
  "items": [
    {
      "type": "contact",
      "resource": "contacts",
      "id": "con_8f2c1a7e-3b4d-4e5f-9a6b-7c8d9e0f1a2b",
      "title": "Acme Soluciones SL",
      "subtitle": "B12345674 · hola@acme.example",
      "displayType": "contact",
      "webUrl": "https://3.facturadirecta.com/company/com_…/contacts/con_8f2c1a7e-3b4d-4e5f-9a6b-7c8d9e0f1a2b"
    },
    {
      "type": "invoice",
      "resource": "invoices",
      "id": "inv_7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "title": "F-2026-0012",
      "subtitle": "Acme Soluciones SL",
      "snippet": "…Consultoría trimestral · Soporte técnico…",
      "date": "2026-03-01",
      "total": 121,
      "currency": "EUR",
      "displayType": "invoice",
      "webUrl": "https://3.facturadirecta.com/company/com_…/doc/inv_7c9e6679-7425-40de-944b-e07fc1f90ae7"
    }
  ],
  "groups": [
    { "type": "contact", "resource": "contacts", "hasMore": false },
    { "type": "invoice", "resource": "invoices", "hasMore": true }
  ]
}
```

| Campo | Significado |
|---|---|
| `items[].type` | Tipo de documento (`contact`, `invoice`, `bill`…). |
| `items[].resource` | Recurso de la API con el que se lee: `GET /{companyId}/<resource>/{id}` (`recurring` es `GET /{companyId}/recurring/invoices/{id}`). |
| `items[].title` | Lo que identifica el elemento: el número del documento, el nombre del contacto, el proveedor de la factura de compra, el empleado de la nómina, el título del producto o de la factura recurrente. |
| `items[].subtitle` | Segunda línea: el contacto del documento, el NIF y el email del contacto, la referencia del producto o de la factura de compra, el título de la nómina. Puede faltar. |
| `items[].snippet` | Fragmento del texto del documento alrededor de la palabra buscada que coincide (una línea, una nota…), para saber por qué ha salido. Solo cuando la coincidencia no está ya en `title` ni en `subtitle`. |
| `items[].date`, `total`, `currency` | Fecha e importe del documento, cuando los tiene. |
| `items[].displayType` | Tipo tal como se muestra: `ticket` en las facturas de compra que son tickets; el tipo en el resto. |
| `items[].webUrl` | Ficha del elemento en la aplicación web (ver [Abrir un resultado en FacturaDirecta](../guides/overview.md#abrir-un-resultado-en-facturadirecta)). |
| `groups[]` | Tipos con algún resultado, en el orden en que aparecen en `items`, y si hay más de los devueltos (`hasMore`). |

## Ejemplo

```bash
curl "https://app.facturadirecta.com/api/$COMPANY_ID/search?q=F-2026-12&types=invoices" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Con el `id` del resultado se lee el documento completo:

```bash
curl "https://app.facturadirecta.com/api/$COMPANY_ID/invoices/inv_7c9e6679-7425-40de-944b-e07fc1f90ae7" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| GET | `/{companyId}/search` | `search` | `bills:read`, `clientOrders:read`, `contacts:read`, `deliveryNotes:read`, `estimates:read`, `invoices:read`, `payrolls:read`, `products:read`, `purchaseOrders:read`, `recurring:read` | Búsqueda global de documentos |

## Scopes

- **`bills:read`** — Lectura de facturas de compra y tickets.
- **`clientOrders:read`** — Lectura de pedidos.
- **`contacts:read`** — Lectura de contactos.
- **`deliveryNotes:read`** — Lectura de albaranes.
- **`estimates:read`** — Lectura de presupuestos.
- **`invoices:read`** — Lectura de facturas.
- **`payrolls:read`** — Lectura de nóminas.
- **`products:read`** — Lectura de productos.
- **`purchaseOrders:read`** — Lectura de órdenes de compra.
- **`recurring:read`** — Lectura de documentos recurrentes.
