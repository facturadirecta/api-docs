---
title: Versiones de documentos
audience:
  - developers
status: draft
---

# Versiones de documentos

Cada vez que se crea, modifica, elimina o recupera un documento, FacturaDirecta
guarda una copia completa de cómo queda. La API expone ese historial como
**versiones** del documento: quién hizo cada cambio, cuándo, qué campos
cambiaron y cómo era el documento en cada momento.

Sirve para auditar los cambios de un documento concreto, para averiguar qué se
cambió por error y para recuperar esos datos, también de un documento ya
eliminado.

## Recursos con versiones

| Recurso | Operaciones | Scope |
|---|---|---|
| Facturas de venta | `GET /{companyId}/invoices/{id}/versions[/{versionId}]` | `invoices:read` |
| Presupuestos | `GET /{companyId}/estimates/{id}/versions[/{versionId}]` | `estimates:read` |
| Albaranes | `GET /{companyId}/deliveryNotes/{id}/versions[/{versionId}]` | `deliveryNotes:read` |
| Pedidos | `GET /{companyId}/clientOrders/{id}/versions[/{versionId}]` | `clientOrders:read` |
| Facturas de compra y tickets | `GET /{companyId}/bills/{id}/versions[/{versionId}]` | `bills:read` |
| Órdenes de compra | `GET /{companyId}/purchaseOrders/{id}/versions[/{versionId}]` | `purchaseOrders:read` |
| Contactos | `GET /{companyId}/contacts/{id}/versions[/{versionId}]` | `contacts:read` |
| Productos | `GET /{companyId}/products/{id}/versions[/{versionId}]` | `products:read` |
| Bancos | `GET /{companyId}/banks/{id}/versions[/{versionId}]` | `banks:read` |
| Métodos de pago | `GET /{companyId}/paymentMethods/{id}/versions[/{versionId}]` | `paymentMethods:read` |
| Nóminas | `GET /{companyId}/payrolls/{id}/versions[/{versionId}]` | `payrolls:read` |

Las versiones exigen el scope de lectura del recurso, como su
`GET /{companyId}/<recurso>/{id}`, y no el de la actividad.

## Qué es una versión

Cada versión corresponde a un registro de la [actividad](../sections/activity.md) del
documento:

- `inserted`: el alta.
- `updated`: una modificación. Su `subtype` indica una acción concreta, como
  `voided` al anular una factura.
- `archived`: la eliminación. La copia es el documento tal como estaba al
  eliminarlo.
- `unarchived`: la recuperación.

El resto de la actividad del documento (envíos, comentarios…) no crea
versiones. El identificador de la versión es el `id` de su registro de
actividad.

La copia es la que se guardó en su momento: la de un documento antiguo puede no
cumplir el esquema actual del recurso.

## Listar las versiones

`GET /{companyId}/<recurso>/{id}/versions` devuelve las versiones de un
documento, eliminado o no, con **paginación por cursor** (ver el patrón B en la
[guía de paginación](./pagination.md)):

- `limit`: entre `1` y `100`. Por defecto, `50`.
- `cursor`: el `nextCursor` de la página anterior, tal cual se recibió. Se
  omite en la primera página.
- `sortBy`: `-timestamp` (por defecto, lo más reciente primero) o `timestamp`.

La respuesta es `{ items, hasMore, nextCursor }`: `nextCursor` es el cursor de
la página siguiente, o `null` cuando no hay más versiones.

Cada elemento lleva `id`, `timestamp`, `type`, `subtype` y `username`. En las
modificaciones añade `changedFields`, con las rutas de los campos que cambiaron
respecto a la versión anterior (como mucho 50; si hay más,
`changedFieldsTruncated` es `true`).

```json
{
  "items": [
    {
      "id": 482913,
      "timestamp": "2026-09-15T08:42:17.351Z",
      "type": "updated",
      "subtype": null,
      "username": "ana.garcia@ejemplo.es",
      "changedFields": ["main.dueDate", "main.lines[1].unitPrice"]
    },
    {
      "id": 480002,
      "timestamp": "2026-09-14T17:05:40.120Z",
      "type": "inserted",
      "subtype": null,
      "username": "ana.garcia@ejemplo.es"
    }
  ],
  "hasMore": false,
  "nextCursor": null
}
```

## Obtener una versión

`GET /{companyId}/<recurso>/{id}/versions/{versionId}` devuelve:

- `content`: la copia del documento en esa versión, en el mismo formato que
  `GET /{companyId}/<recurso>/{id}`.
- `tags`: sus etiquetas en esa versión, si se guardaron.
- `version`: los datos de la versión (`id`, `timestamp`, `type`, `subtype`,
  `username`).
- `previousVersionId` y `changes`: en una modificación, la versión anterior y
  cada campo que cambió, con su valor antes y después. En el resto de
  versiones, `null` y `[]`.

```json
{
  "content": { "uuid": "inv_7c1e2f40-5a8b-4d3c-9e6f-2b1a0c9d8e7f", "type": "invoice", "main": { "...": "..." } },
  "tags": ["mayorista"],
  "version": { "id": 482913, "timestamp": "2026-09-15T08:42:17.351Z", "type": "updated", "subtype": null, "username": "ana.garcia@ejemplo.es" },
  "previousVersionId": 480002,
  "changes": [
    { "path": "main.dueDate", "before": "2026-10-15", "after": "2026-11-15" },
    { "path": "main.lines[1].unitPrice", "before": 120, "after": 12 }
  ]
}
```

Sobre `changes` (y `changedFields`):

- Se compara lo que edita el usuario: `main`, los adjuntos (`attachments`) y
  las etiquetas (`tags`). `accounting`, `books`, `meta` e `import` los rellena
  el servidor al guardar (el asiento, los libros de impuestos, los datos de
  gestión y los de la importación) y no se comparan, aunque están completos en
  `content`.
- Algunos campos de `main` también se recalculan al guardar, como los totales,
  los impuestos o el total de cada línea: al cambiar un precio aparecen junto
  al precio.
- `path` es la ruta del campo: `main.dueDate`, `main.lines[2].unitPrice`,
  `attachments` o `tags`.
- Si el campo no existía en la versión anterior, falta `before`; si ya no
  existe, falta `after`.
- Las listas se comparan por posición: una línea nueva al final aparece como
  `main.lines[3]` con el valor completo en `after`, y una línea quitada, con el
  valor completo en `before`.

## Desde la actividad

El [registro de actividad](../sections/activity.md) dice qué documentos se modificaron,
quién y cuándo (`type=updated`), pero no qué cambió. El `id` de cada registro
es también el de su versión: con él pides la versión al recurso del documento
(`document.type` y `document.id`) para ver los campos cambiados y sus valores.

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/activity?documentType=invoice&type=updated"
```

## Recuperar datos de una versión

La API no tiene una operación de «restaurar»: los datos se recuperan con la
operación de modificación del recurso, que aplica las mismas validaciones y
restricciones que cualquier otro cambio.

1. Localiza el cambio en las versiones del documento: `changedFields` dice qué
   campos tocó cada modificación.
2. Pide esa versión: `changes` trae el valor anterior de cada campo, y
   `previousVersionId`, la versión completa anterior al cambio.
3. Pide el documento actual con `GET /{companyId}/<recurso>/{id}`.
4. Envía el documento actual con los campos recuperados a la operación de
   modificación (`PUT /{companyId}/<recurso>/{id}`). Cambia solo los campos que
   quieres recuperar, en lugar de reenviar la copia entera: puede ser de un
   esquema anterior y lleva campos que se recalculan, como los totales.

Ten en cuenta que:

- Una factura ya enviada a VeriFactu o TicketBAI no se puede modificar: se
  corrige con una [rectificativa](./invoices-rectificativas.md).
- Los cambios recalculan lo que dependa del documento, como su contabilidad o el
  stock.
- Un documento eliminado no se recupera por la API: créalo de nuevo con los
  datos de la copia, sin su `uuid`. Será un documento nuevo, con otro
  identificador.

###### Copy as cURL

Versiones de una factura:

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/invoices/inv_7c1e2f40-5a8b-4d3c-9e6f-2b1a0c9d8e7f/versions"
```

Una versión, con sus cambios:

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/invoices/inv_7c1e2f40-5a8b-4d3c-9e6f-2b1a0c9d8e7f/versions/482913"
```

## Permisos

- El scope de lectura del recurso, con los mismos requisitos de rol que su
  `GET /{companyId}/<recurso>/{id}`.
- En los contactos, solo los de las facetas (clientes, proveedores, empleados)
  que permite tu rol: el resto responde `404`.
- Los IBAN de bancos y métodos de pago se ven completos solo con
  `banks:readIban`; si no, como en el resto de la API, solo sus últimos cuatro
  dígitos (`iban4`).

## Errores comunes

- `404 Not Found`: el documento no existe, no es de ese recurso o no lo puedes
  ver, o la versión no es de ese documento.
- `400 ValidationError`: `versionId` no es un número, `cursor` no es un cursor
  devuelto por la API o `limit` está fuera de `1`–`100`.
- `403 Forbidden`: falta el scope de lectura del recurso.
- `422 Unprocessable Entity` (`code: "activity_query_timeout"`): la consulta ha
  tardado demasiado. Inténtalo con un `limit` menor.

Ver [Errores y validaciones](./errors.md) para el formato general.
