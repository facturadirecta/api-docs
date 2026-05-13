---
title: Contactos
audience:
  - developers
status: draft
---

# Contactos

Un contacto representa una persona física o empresa con la que mantienes una
relación contable: clientes a los que facturas, proveedores de los que recibes
gastos y empleados a los que pagas nóminas.

En FacturaDirecta un mismo contacto puede actuar simultáneamente como cliente,
proveedor y empleado. No son tipos excluyentes: son **facetas** que se activan
al asociar al contacto la cuenta contable correspondiente.

## Facetas: cliente, proveedor, empleado

Las facetas se controlan a través del objeto `content.main.accounts`:

- `accounts.client` → habilita al contacto como cliente (cuenta 430* por defecto).
- `accounts.provider` → habilita al contacto como proveedor (cuenta 400/410*).
- `accounts.employee` → habilita al contacto como empleado (cuenta 465*).

Cada faceta principal tiene su contraparte de anticipos
(`clientCredit`, `providerCredit`, `employeeCredit`). Si indicas solo la cuenta
principal con el prefijo esperado, FacturaDirecta autocompleta la de anticipos
con el valor por defecto de la empresa.

**Comportamiento por defecto al crear un contacto.** Si no envías `accounts`,
el contacto se crea como cliente con las cuentas contables por defecto. Para
crear un proveedor o un empleado debes indicarlo explícitamente.

Para listar contactos por faceta, usa los filtros `isClient`, `isProvider` o
`isEmployee` en `GET /contacts`. Devuelven los contactos que tienen la cuenta
correspondiente asignada.

## Operaciones

- [Lista de contactos](#lista-de-contactos)
- [Crear contacto](#crear-contacto)
- [Obtener un contacto](#obtener-un-contacto)
- [Actualizar contacto](#actualizar-contacto)
- [Borrar contacto](#borrar-contacto)
- [Actualizar etiquetas de contacto](#actualizar-etiquetas-de-contacto)

## Lista de contactos

`GET /{companyId}/contacts` devuelve los contactos de la empresa indicada,
paginados.

**Parámetros de consulta:**

- `search` — búsqueda libre por nombre, apellidos o nombre comercial.
- `fiscalId` — identificador fiscal exacto (sin separadores).
- `vatEU` — identificador IVA intracomunitario.
- `externalId` — referencia externa asignada al importar.
- `phone` — teléfono exacto.
- `country` — código país ISO 3166-1 Alpha-2 (`ES`, `FR`...).
- `isClient`, `isProvider`, `isEmployee` — facetas (ver sección superior).
- `allTheseTags`, `anyOfTheseTags`, `hasTags` — filtros por etiquetas.
- `sortBy` — campo de orden.
- `related` — recursos relacionados a expandir.

Ver [Paginación](../guides/pagination.md) para los parámetros comunes
(`offset`, `limit`, `hasMore`).

###### Ejemplo de respuesta JSON

```json
{
  "pagination": { "offset": 0, "limit": 50, "total": 124 },
  "items": [
    {
      "content": {
        "type": "contact",
        "uuid": "con_2c4f8e21-7b3a-4d9c-9e1f-a8b7c6d5e4f3",
        "main": {
          "title": "Acme Diseño",
          "name": "Acme Diseño S.L.",
          "email": "facturacion@acme.es",
          "phone": "+34 932 000 000",
          "address": "Carrer Aribau 123",
          "zipcode": "08036",
          "city": "Barcelona",
          "region": "Barcelona",
          "country": "ES",
          "fiscalId": "B12345674",
          "fiscalIdCountry": "ES",
          "accounts": {
            "client": "4300001",
            "clientCredit": "4380001"
          },
          "currency": "EUR"
        }
      },
      "tags": ["agencia", "barcelona"],
      "creationDate": "2025-11-14T10:22:31.000Z",
      "modificationDate": "2026-04-02T08:15:00.000Z"
    }
  ]
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/contacts?isClient=true&country=ES&limit=50"
```

## Crear contacto

`POST /{companyId}/contacts` crea un contacto en la empresa indicada.

**Parámetros del body:**

- `content.type` — siempre `"contact"`.
- `content.main` — campos del contacto.
- `tags` (opcional) — lista de etiquetas a asociar.

**Notas:**

- Si omites `accounts`, el contacto se crea como cliente con las cuentas por defecto de la empresa.
- `fiscalId` se valida cuando `fiscalIdCountry` es `ES` (o `country` es `ES` si no se indica `fiscalIdCountry`).
- La respuesta es el contacto creado completo, en el mismo formato que
  [Obtener un contacto](#obtener-un-contacto).

###### Ejemplo de request JSON

```json
{
  "content": {
    "type": "contact",
    "main": {
      "name": "Acme Diseño S.L.",
      "email": "facturacion@acme.es",
      "fiscalId": "B12345674",
      "address": "Carrer Aribau 123",
      "zipcode": "08036",
      "city": "Barcelona",
      "country": "ES"
    }
  },
  "tags": ["agencia", "barcelona"]
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{
    "content": {
      "type": "contact",
      "main": {
        "name": "Acme Diseño S.L.",
        "email": "facturacion@acme.es",
        "fiscalId": "B12345674",
        "address": "Carrer Aribau 123",
        "zipcode": "08036",
        "city": "Barcelona",
        "country": "ES"
      }
    },
    "tags": ["agencia", "barcelona"]
  }' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/contacts"
```

## Obtener un contacto

`GET /{companyId}/contacts/{id}` devuelve un contacto por su ID.

**Parámetros:**

- `related` (opcional) — recursos relacionados a expandir, por ejemplo `invoices,bills`.

###### Ejemplo de respuesta JSON

```json
{
  "content": {
    "type": "contact",
    "uuid": "con_2c4f8e21-7b3a-4d9c-9e1f-a8b7c6d5e4f3",
    "main": {
      "title": "Acme Diseño",
      "name": "Acme Diseño S.L.",
      "email": "facturacion@acme.es",
      "phone": "+34 932 000 000",
      "address": "Carrer Aribau 123",
      "zipcode": "08036",
      "city": "Barcelona",
      "region": "Barcelona",
      "country": "ES",
      "fiscalId": "B12345674",
      "fiscalIdCountry": "ES",
      "accounts": {
        "client": "4300001",
        "clientCredit": "4380001"
      },
      "currency": "EUR"
    }
  },
  "tags": ["agencia", "barcelona"],
  "creationDate": "2025-11-14T10:22:31.000Z",
  "modificationDate": "2026-04-02T08:15:00.000Z",
  "related": {}
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/contacts/con_2c4f8e21-7b3a-4d9c-9e1f-a8b7c6d5e4f3"
```

## Actualizar contacto

`PUT /{companyId}/contacts/{id}` sustituye **el contenido completo** del
contacto. No es un PATCH: lo que no envíes, se borra.

**Parámetros del body:** mismo formato que [Crear contacto](#crear-contacto).
Para añadir una faceta a un contacto existente (por ejemplo, convertir un
cliente en cliente-y-proveedor) reenvía el contacto completo con las cuentas
adicionales en `accounts`.

**Respuesta:** el contacto actualizado completo, en el mismo formato que
[Obtener un contacto](#obtener-un-contacto).

###### Ejemplo de request JSON

```json
{
  "content": {
    "type": "contact",
    "main": {
      "name": "Acme Diseño S.L.",
      "email": "facturacion@acme.es",
      "phone": "+34 932 000 000",
      "fiscalId": "B12345674",
      "address": "Carrer Aribau 123",
      "zipcode": "08036",
      "city": "Barcelona",
      "country": "ES",
      "accounts": {
        "client": "4300001",
        "clientCredit": "4380001",
        "provider": "4100001",
        "providerCredit": "4070001"
      }
    }
  }
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '{
    "content": {
      "type": "contact",
      "main": {
        "name": "Acme Diseño S.L.",
        "email": "facturacion@acme.es",
        "phone": "+34 932 000 000",
        "fiscalId": "B12345674",
        "address": "Carrer Aribau 123",
        "zipcode": "08036",
        "city": "Barcelona",
        "country": "ES",
        "accounts": {
          "client": "4300001",
          "clientCredit": "4380001",
          "provider": "4100001",
          "providerCredit": "4070001"
        }
      }
    }
  }' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/contacts/con_2c4f8e21-7b3a-4d9c-9e1f-a8b7c6d5e4f3"
```

## Borrar contacto

`DELETE /{companyId}/contacts/{id}` elimina un contacto de la empresa.

**Restricciones.** Si el contacto tiene documentos asociados (facturas,
gastos, nóminas), la API rechaza el borrado con `409 Conflict` y un mensaje
indicando el motivo. En ese caso conviene archivar el contacto o reasignar sus
documentos antes de borrarlo.

**Respuesta:** `{ "result": true }`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X DELETE \
  "https://app.facturadirecta.com/api/$COMPANY_ID/contacts/con_2c4f8e21-7b3a-4d9c-9e1f-a8b7c6d5e4f3"
```

## Actualizar etiquetas de contacto

`PUT /{companyId}/contacts/{id}/tags` reemplaza el conjunto de etiquetas de un
contacto sin tocar el resto del contenido. Útil cuando solo quieres reclasificar
sin reenviar la estructura completa del contacto.

**Parámetros del body:**

- `tags` — lista completa de etiquetas (sustituye, no añade).

###### Ejemplo de request JSON

```json
{
  "tags": ["agencia", "premium", "barcelona"]
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -X PUT \
  -d '{"tags":["agencia","premium","barcelona"]}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/contacts/con_2c4f8e21-7b3a-4d9c-9e1f-a8b7c6d5e4f3/tags"
```

## Identificador fiscal

`fiscalId` es obligatorio para emitir facturas a clientes españoles. Se valida
formato (NIF, CIF o NIE) y se almacena sin separadores ni espacios. Si no
indicas `fiscalIdCountry`, se asume el mismo país que la dirección fiscal del
contacto.

Para contactos extracomunitarios sin identificador fiscal válido, deja el
campo vacío. Para contactos intracomunitarios, usa el campo `vatEU` (formato
con prefijo de país: `ESB12345674`).

## Errores comunes

- `400 ValidationError` — `fiscalId` con formato inválido para el país indicado.
- `409 Conflict` — borrado de contacto con documentos asociados.
- `409 Conflict` — `fiscalId` duplicado en la empresa.

Ver [Errores y validaciones](../guides/errors.md) para el formato general de
respuestas de error.

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| GET | `/{companyId}/contacts` | `getContacts` | `contacts:read` | Lista de contactos |
| GET | `/{companyId}/contacts/{id}` | `getContact` | `contacts:read` | Obtener un contacto |
| POST | `/{companyId}/contacts` | `createContact` | `contacts:write` | Crear contacto |
| PUT | `/{companyId}/contacts/{id}` | `updateContact` | `contacts:write` | Actualizar contacto |
| PUT | `/{companyId}/contacts/{id}/tags` | `updateContactTags` | `contacts:write` | Actualizar etiquetas de contacto |
| DELETE | `/{companyId}/contacts/{id}` | `deleteContact` | `contacts:write` | Borrar contacto |

## Scopes

- **`contacts:read`** — Lectura de contactos.
- **`contacts:write`** — Modificación de contactos.
