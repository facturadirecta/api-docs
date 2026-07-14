---
title: Inicio rápido con la API
description: >-
  Haz tu primera llamada a la API de FacturaDirecta en minutos. Guía paso a paso con ejemplos de
  cURL.
audience:
  - developers
status: draft
---

# Inicio rápido con la API

<!-- source: api.ts -->

Esta guía te ayuda a hacer tu primera llamada a la API pública de
FacturaDirecta. En pocos pasos tendrás una conexión funcional usando una API key.

> **Recomendación**: haz tus primeras pruebas en un [entorno de pruebas
> (sandbox)](https://help.facturadirecta.com/es/articles/15157082-entorno-de-pruebas-sandbox)
> antes de usar la API en producción.

## Requisitos previos

Necesitas:

- Una cuenta en FacturaDirecta.
- Una empresa real o sandbox.
- El **companyId** de esa empresa.
- Una **API key** con los permisos necesarios.

Si todavía no tienes la clave o el `companyId`, consulta [API keys en
Ajustes](https://help.facturadirecta.com/es/articles/15157077-api-keys).

## Variables para los ejemplos

Guarda estos valores en variables de entorno para copiar los ejemplos con menos
riesgo de error:

```shell
export fd_company_id="com_tu_empresa"
export fd_api_key="tu_api_key"
```

La URL base de producción es:

```shell
export fd_api_base="https://app.facturadirecta.com/api"
```

## Primera llamada: listar facturas

<!-- source: apiInvoices -->

Vamos a listar facturas de venta:

```shell
curl -s \
  -H "facturadirecta-api-key: $fd_api_key" \
  "$fd_api_base/$fd_company_id/invoices?limit=5"
```

Si todo va bien, recibirás una respuesta JSON. Las listas paginadas devuelven
normalmente `pagination` e `items`:

```json
{
  "pagination": {
    "offset": 0,
    "limit": 5,
    "total": 42
  },
  "items": [
    {
      "uuid": "inv_...",
      "main": {
        "date": "2026-01-15",
        "docNumber": { "series": "F-##-", "number": 1 },
        "currency": "EUR"
      }
    }
  ]
}
```

> El contenido exacto de cada factura depende del schema público `Invoice`.
> Consulta [Facturas de venta](../sections/invoices.md) para el detalle del recurso.

## Probar un recurso más simple: contactos

<!-- source: apiContacts -->

Si tu empresa todavía no tiene facturas, puedes probar con contactos:

```shell
curl -s \
  -H "facturadirecta-api-key: $fd_api_key" \
  "$fd_api_base/$fd_company_id/contacts?limit=5"
```

Para leer un contacto concreto, usa su ID:

```shell
curl -s \
  -H "facturadirecta-api-key: $fd_api_key" \
  "$fd_api_base/$fd_company_id/contacts/con_..."
```

## Crear una factura de prueba

<!-- source: apiInvoices -->

Cuando ya puedas leer datos, prueba una operación de escritura en sandbox. Este
ejemplo crea una factura completa mínima.

Crea un fichero `invoice.json`:

```json
{
  "content": {
    "main": {
      "contact": "con_2e3f4a18-9b7c-4d6a-8e1f-5c2d3b4a7e9f",
      "date": "2026-05-13",
      "dueDate": "2026-06-12",
      "currency": "EUR",
      "exchangeRate": 1,
      "docNumber": { "series": "F-##-" },
      "fiscalPosition": "general",
      "account": "700000",
      "theme": "default",
      "draft": true,
      "voided": false,
      "simplified": false,
      "lines": [
        {
          "account": "700000",
          "quantity": 1,
          "unitPrice": 100,
          "discount": 0,
          "discountRate": 0,
          "lineTotal": 100,
          "tax": ["S_IVA_21"],
          "text": "Servicios de consultoría"
        }
      ]
    }
  }
}
```

Envíalo a la API:

```shell
curl -s \
  -H "facturadirecta-api-key: $fd_api_key" \
  -H "Content-Type: application/json" \
  -d @invoice.json \
  "$fd_api_base/$fd_company_id/invoices"
```

Notas:

- Usa `draft: true` para que la factura sea provisional durante las pruebas.
- Sustituye `contact`, `theme`, `account`, `tax` y la serie por valores válidos
  de tu empresa.
- Si tu empresa usa TicketBAI o VeriFactu, revisa las guías específicas antes de
  crear facturas definitivas.

## Errores comunes

### 401 Unauthorized

La API key no es válida o no se ha enviado. Comprueba:

- que la cabecera se llama `facturadirecta-api-key`;
- que la clave está copiada completa;
- que la clave no ha sido eliminada.

### 403 Forbidden

La clave existe, pero no tiene permisos para la operación. Comprueba:

- que incluye el scope necesario, por ejemplo `invoices:read` para leer facturas;
- que el `companyId` corresponde a la empresa donde se creó la API key.

### 404 Not Found

El recurso no existe o la URL no es correcta. Comprueba:

- que el `companyId` es correcto;
- que el path del endpoint existe;
- que el ID del recurso (`inv_...`, `con_...`) pertenece a esa empresa.

## Siguientes pasos

- Consulta la [API Pública de FacturaDirecta](https://help.facturadirecta.com/es/articles/13538823-api-publica-de-facturadirecta) para ver la visión
  general.
- Revisa [Autenticación](./authentication.md) para API keys y OAuth2.
- Revisa [Paginación y filtros estándar](./pagination.md).
- Explora la [referencia OpenAPI](https://www.facturadirecta.com/api).
- Configura [webhooks](https://help.facturadirecta.com/es/articles/13905261-webhooks) si tu integración necesita recibir cambios
  en tiempo real.
