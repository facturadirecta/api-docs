---
title: Flujo de pedidos y órdenes de compra
audience:
  - developers
status: draft
---

# Flujo de pedidos y órdenes de compra

El **pedido de cliente** (`clientOrder`) registra un encargo en firme de un
cliente antes de su entrega y facturación. La **orden de compra**
(`purchaseOrder`) registra un encargo en firme a un proveedor antes de la
recepción de la mercancía. FacturaDirecta conecta ambos documentos **a nivel
de línea**: una línea de pedido puede abastecerse con una línea de orden de
compra, y sus estados se coordinan automáticamente.

Esta guía cubre:

- Los **estados de línea** de cada documento y cómo se deriva el estado del
  documento.
- El **vínculo línea-a-línea** entre pedido y orden.
- La **propagación de estados** cuando se guarda o se borra una orden.
- Cómo **generar una orden de compra** a partir de líneas de pedido pendientes.

Los dos documentos forman parte del **Módulo Inventario**: están incluidos en
el plan **Diamante** y se pueden contratar como módulo en **Bronce**, **Plata**
y **Oro**. Sin esa disponibilidad, cualquier escritura sobre ellos responde
`403 Forbidden`.

## Estados de línea

Cada línea tiene su propio estado, y el estado del documento se **calcula** a
partir de los estados de sus líneas. Las líneas vacías (sin texto ni importe)
no cuentan, y un documento sin líneas con contenido queda en `pending`.

**Línea de pedido de cliente** — `pending` (pendiente), `ordered` (pedida a
proveedor), `ready` (lista para entregar), `delivered` (entregada) o `canceled`
(cancelada). Estado del pedido, por prioridad: `pending` si alguna línea está
`pending` u `ordered`; si no `ready` si alguna está `ready`; si no `delivered`
si alguna está `delivered`; si no `canceled`.

**Línea de orden de compra** — `pending` (pendiente de recibir), `received`
(recibida) o `canceled`. Estado de la orden, por la misma lógica de prioridad:
`pending` > `received` > `canceled`.

En ambos documentos, `main.state` se recalcula en cada guardado: el valor que
envíes se descarta.

> El estado `ordered` de la línea de pedido lo fija FacturaDirecta cuando la
> línea se vincula a una orden de compra: no lo asignes manualmente.

## Vínculo línea-a-línea

El vínculo es **bidireccional, 1:1 y exclusivo**. Cada línea tiene un `id`
estable que la identifica dentro del documento; los vínculos referencian ese
`id`:

- Línea de pedido: `purchaseOrder` (id de la orden) + `purchaseOrderLine` (id
  de la línea de orden). Además `provider` (proveedor por línea, usado para
  generar la orden).
- Línea de orden: `clientOrder` (id del pedido) + `clientOrderLine` (id de la
  línea de pedido). Además `client` (cliente final, trazabilidad).

El vínculo se establece **desde la orden de compra**: al crear o guardar una
orden cuyas líneas indican `clientOrder`/`clientOrderLine`, FacturaDirecta
completa el vínculo recíproco en el pedido. Reglas validadas al guardar:

- Los dos campos del vínculo (documento + línea) deben indicarse **juntos**.
- Una línea de pedido no puede estar vinculada a **dos** líneas de orden
  (exclusividad).
- El `id` de línea debe ser **único** dentro del documento.
- La línea de pedido referenciada debe existir.
- El **proveedor** de la línea de pedido debe coincidir con el proveedor de la
  orden de compra.
- El **producto** y la **cantidad** deben coincidir en las dos líneas. Si
  necesitas comprar unidades adicionales, añade una línea de reposición
  independiente en la orden en vez de cambiar la cantidad.

Desde el **pedido** no se puede tocar nada de eso: intentar crear, cambiar o
borrar el vínculo, o cambiar el producto, la cantidad o el proveedor de una
línea ya vinculada, devuelve `400`. Tampoco se puede borrar un pedido con
líneas vinculadas a órdenes activas.

> **Identidad de línea en `PUT`.** Al actualizar un documento por API, las
> líneas se sustituyen por las del cuerpo. Una línea que trae un `id` existente
> **conserva su identidad** (y sus vínculos); una línea sin `id` es nueva.
> Regenerar los `id` en cada `PUT` rompería los vínculos.

## Propagación de estados

Al guardar o borrar una orden de compra, el estado de sus líneas se propaga a
las líneas de pedido vinculadas (la orden manda; el pedido es el afectado).
Cada pedido afectado se guarda con su propio historial y emite el webhook
`client_order.updated`.

| Estado línea **orden** | Estado línea **pedido** | Resultado |
|---|---|---|
| `pending` | `pending` o `ready` | pedido → `ordered` |
| `received` | `pending` u `ordered` | pedido → `ready` |
| `canceled` | `pending` u `ordered` | pedido → `pending` y se rompe el vínculo |

Los estados `delivered` y `canceled` del pedido son **terminales** frente a la
propagación: la recepción o la cancelación de la mercancía no los pisa.
Al **borrar** la orden, las líneas de pedido `ordered` y `ready` vuelven a
`pending` y se desvinculan; al **recuperarla**, se re-vinculan solo si las
líneas de pedido siguen libres.

En sentido contrario no hay propagación: los estados `pending`, `ordered` y
`ready` de una línea de pedido **vinculada** son propiedad del servidor. Si
guardas el pedido con uno distinto del que hay almacenado, el servidor restaura
el suyo en silencio, sin error. Esto evita que un `PUT` construido sobre una
copia anterior deshaga una recepción ya confirmada. `delivered` y `canceled`
sí los decides tú desde el pedido.

## Generar una orden de compra desde pedidos

El flujo típico de compras bajo pedido:

1. Asigna a cada línea de pedido un **proveedor** (`provider`). Si el producto
   de la línea tiene un proveedor habitual (`purchases.provider` del producto),
   ese es el valor por defecto que propone la interfaz. El producto debe tener
   activada su faceta de compra.
2. Crea una **orden de compra** con ese proveedor en `main.contact` y con las
   líneas de los pedidos correspondientes, indicando en cada línea
   `clientOrder` y `clientOrderLine` (el `id` de la línea de pedido de origen).
3. Al **guardar** la orden, las líneas de pedido vinculadas pasan a `ordered`.
4. Cuando se recibe la mercancía, cambia el estado de las líneas de orden a
   `received`: las líneas de pedido pasan a `ready`.
5. Entrega al cliente y marca esas líneas de pedido como `delivered`.

Facturar un pedido (convertirlo en factura o albarán) **no** cambia los estados
de línea: la entrega y la recepción son actos explícitos.

## Concurrencia

Las escrituras de órdenes de compra de una misma empresa se **serializan**,
porque cada una puede tocar los pedidos que abastece. Si ya hay 20 escrituras
de órdenes en curso en la misma empresa, la siguiente responde
`429 Too Many Requests`. Carga las órdenes en serie o con poca concurrencia, y
reintenta ante un `429`.

## Efecto sobre el stock

Con el control de stock activo, las líneas con producto de estos documentos
alimentan magnitudes que se consultan en el
[recurso de productos](../sections/products.md):

- Las líneas de **pedido de cliente** activas suman al stock `committed`
  (comprometido).
- Las líneas de **orden de compra** pendientes de recibir suman al stock
  `incoming` (previsto).

El **stock físico** lo mueve el documento que la empresa haya elegido como
disparador. En compras, la opción por defecto es la propia orden: marcar una
línea como `received` da entrada a la mercancía. En ventas, la opción por
defecto es el albarán, pero la empresa puede elegir que muevan la factura o la
propia línea de pedido marcada como `delivered`. Esa elección se configura en
la interfaz, no por API, y la deduplicación por cadena de documentos garantiza
que una misma mercancía solo se descuenta una vez.

El almacén afectado es el de `main.warehouse` del documento, o el almacén por
defecto de la empresa si no se indica. Si la empresa tiene configurado bloquear
los negativos, un documento que dejaría el stock físico por debajo de cero se
rechaza al guardar.

## Scopes

- `clientOrders:read` / `clientOrders:write` — pedidos de cliente.
- `purchaseOrders:read` / `purchaseOrders:write` — órdenes de compra.
