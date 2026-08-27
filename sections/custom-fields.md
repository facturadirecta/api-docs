---
title: Campos personalizados
audience:
  - developers
status: draft
---

# Campos personalizados

El recurso `customFields` permite descubrir y gestionar las definiciones de
campos personalizados de una empresa. Los valores se guardan en los documentos
dentro de `content.main.customFields`, usando como clave el `id` de la
definición, no su título visible.

```json
{
  "content": {
    "main": {
      "customFields": {
        "cfi_1f6f2ec8-7c2b-4c71-9c4f-2257ed34878a": "PO-2026-0042"
      }
    }
  }
}
```

La API valida las claves al crear o actualizar facturas, presupuestos,
albaranes, gastos, nóminas, contactos y facturas recurrentes. Una clave debe
corresponder a una definición existente. Las definiciones borradas siguen
siendo válidas para conservar y editar documentos históricos; una clave
desconocida produce `400 Bad Request`.

## Representación

Todas las operaciones que devuelven un campo usan estos atributos:

| Atributo | Tipo | Descripción |
| --- | --- | --- |
| `id` | string | `cfi_` seguido de UUID v4 en minúsculas. |
| `title` | string | Nombre visible. |
| `docTypes` | string[] | Uno o más de `invoice`, `estimate`, `deliveryNote`, `bill`, `payroll`, `contact`. |
| `archived` | boolean | `true` si el campo está borrado. |

Las respuestas de recurso individual envuelven la representación en
`content`; el listado la devuelve en `items`.

Todas las operaciones reciben `companyId` en el path y aceptan el header
`accept-version`. Las operaciones sobre un campo concreto reciben además su
`id` en el path. Ver [Autenticación](../guides/authentication.md) para el uso de
OAuth, API keys y versionado.

## Listar campos

`GET /{companyId}/customFields` devuelve solo campos activos por defecto.
Admite búsqueda aproximada por `title`, uno o varios `docTypes` con semántica OR,
`archived=false|true|all`, `minCreationDate`, `maxCreationDate`,
`minModificationDate`, `maxModificationDate`, `sortBy`, `limit` y `offset`. Ver
[Paginación](../guides/pagination.md).

```bash
curl -s \
  -H "facturadirecta-api-key: $API_KEY" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/customFields?docTypes=invoice&archived=all"
```

## Obtener un campo

`GET /{companyId}/customFields/{id}` devuelve también campos borrados. Esto
permite resolver los IDs presentes en documentos históricos.

## Crear un campo

`POST /{companyId}/customFields` recibe `title`, `docTypes` y un `id` opcional.
Si omites el ID, FacturaDirecta genera uno. Si lo envías, debe ser exactamente
`cfi_` + UUID v4 en minúsculas. Puedes reutilizar el mismo ID al crear la misma
definición en sandbox y producción.

```json
{
  "id": "cfi_1f6f2ec8-7c2b-4c71-9c4f-2257ed34878a",
  "title": "Referencia de pedido",
  "docTypes": ["invoice", "estimate"]
}
```

Un ID existente devuelve `409 Conflict`, también cuando pertenece a un campo
borrado: borrar no libera el identificador. La creación puede devolver `403`
si el plan no permite campos personalizados o ya se alcanzó su máximo (20 en
los planes que incluyen la función).

## Actualizar un campo

`PUT /{companyId}/customFields/{id}` reemplaza `title` y `docTypes`. No permite
editar un campo borrado: recupéralo primero. Si retiras un tipo de documento,
los valores ya guardados se conservan en su JSON, pero dejan de mostrarse en
ese tipo.

## Borrar un campo

`DELETE /{companyId}/customFields/{id}` realiza un borrado no destructivo. La
definición queda con `archived: true` y los valores de documentos existentes se
conservan.

## Recuperar un campo

`POST /{companyId}/customFields/{id}/restore` no recibe body y es idempotente:
si el campo ya está activo devuelve su estado actual. La recuperación requiere
la función de plan para recuperar documentos borrados y cuenta contra el máximo
de campos activos, por lo que puede devolver `403`.

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| GET | `/{companyId}/customFields` | `getCustomFields` | `customFields:read` | Lista de campos personalizados |
| GET | `/{companyId}/customFields/{id}` | `getCustomField` | `customFields:read` | Obtener campo personalizado |
| POST | `/{companyId}/customFields` | `createCustomField` | `customFields:write` | Crear campo personalizado |
| POST | `/{companyId}/customFields/{id}/restore` | `restoreCustomField` | `customFields:write` | Recuperar campo personalizado |
| PUT | `/{companyId}/customFields/{id}` | `updateCustomField` | `customFields:write` | Actualizar campo personalizado |
| DELETE | `/{companyId}/customFields/{id}` | `deleteCustomField` | `customFields:write` | Borrar campo personalizado |

## Scopes

- **`customFields:read`** — Lectura de campos personalizados.
- **`customFields:write`** — Gestión de campos personalizados.

Para la referencia exhaustiva de campos del body y respuesta, consulta el
[Swagger UI](https://www.facturadirecta.com/api) o el
[openapi crudo](https://app.facturadirecta.com/openapi.json).
