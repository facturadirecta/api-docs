---
title: Configuración de la empresa
audience:
  - developers
status: draft
---

# Configuración de la empresa

Bajo `/settings/...` vive la configuración de la empresa que la API
pública expone como **solo lectura**: la ficha de la empresa y su forma
de trabajar, los impuestos disponibles, las series de numeración por tipo
de documento y las plantillas de impresión. La modificación de esta
configuración se hace desde la interfaz, no por API.

## Las 5 operaciones

| Endpoint | Cubre |
|---|---|
| `GET /settings/company` | Ficha de la empresa y su forma de trabajar: dirección fiscal, moneda, idioma, modo borrador, régimen fiscal, VeriFactu y TicketBAI, bloqueo contable, canales de entrada de la bandeja y módulos disponibles. |
| `GET /settings/taxes/sales` | Catálogo de impuestos para documentos de venta. Documentado en la [guía de Impuestos](../guides/taxes.md). |
| `GET /settings/taxes/purchases` | Catálogo de impuestos para documentos de compra. Documentado en la [guía de Impuestos](../guides/taxes.md). |
| `GET /settings/series/{documentType}` | Series de numeración configuradas en la empresa para un tipo de documento. |
| `GET /settings/themes` | Plantillas de impresión disponibles. |

Las cinco requieren scope **`settings:read`**.

> Para los catálogos de impuestos consulta directamente la
> [guía de Impuestos](../guides/taxes.md) — cubre el modelo completo
> (dos catálogos, schema, autorepercutidos, `lineAmountIsTax`...).
> Esta página documenta las tres operaciones restantes (`company`,
> `series` y `themes`).

## Operaciones

- [Ficha de la empresa](#ficha-de-la-empresa)
- [Lista de series de numeración](#lista-de-series-de-numeración)
- [Lista de plantillas](#lista-de-plantillas)

## Ficha de la empresa

`GET /{companyId}/settings/company` devuelve los datos de la empresa y su
configuración operativa. Es la llamada recomendada justo después de
[`GET /profile`](./profile.md): el perfil dice **qué empresas** hay y esta
operación dice **cómo trabaja** cada una, lo que un integrador o un agente
necesita saber antes de crear documentos o enviar ficheros a la
[bandeja](./inbox.md).

**Parámetros globales aceptados:** `accept-version`.

**Respuesta:** `{ content: CompanySettings }`.

Bloques de `content`:

| Bloque | Qué contiene |
|---|---|
| `company` | Identificación y datos fiscales: `id`, `sandbox`, `name`, `surname`, `brand`, `taxCode`, `email`, `phone`, `website`, `address`, `country`, `businessType` (`EM` empresa, `AU` autónomo, `AS` asociación), `taxRegion`, `currency` y `creationDate`. |
| `locale` | `timezone`, `language` y los formatos de fecha y hora con los que la empresa ve la interfaz y los PDF. La API siempre intercambia fechas en ISO 8601. |
| `sales` | `newInvoiceMode` (`draft` o `final`: estado con el que nacen las facturas si no indicas `content.main.draft`), `taxCalculationMode` (`total` o `lines`) y `externalInvoicesEnabled`. |
| `expenses` | `newBillMode` (igual que `newInvoiceMode`, para gastos) y `taskToExpenseDefaultDate`. |
| `taxes` | `defaultSales` (`BC` bienes o `SV` servicios), `equivalenceSurcharge`, `applyVatEUToFinalConsumer` y `vatEU` (NIF-IVA intracomunitario o `null`). El catálogo de impuestos está en `/settings/taxes/*`. |
| `numbering` | `zeros` y `position` del número de documento. Las series están en `/settings/series/{documentType}`. |
| `accounting` | `startDate` y `lockDate`: no se pueden crear ni modificar documentos con fecha igual o anterior al bloqueo. |
| `fiscal` | `verifactu` (`enabled`, `environment`, `mode`, `startDate`, `endDate`, `exemption`) y `ticketbai` (`enabled`, `environment`, `region`, `startDate`). Sin certificados. |
| `email` | `from`: remitentes disponibles para enviar documentos. |
| `stock` | Solo si la empresa tiene control de stock: `salesTrigger`, `purchasesTrigger`, `negativesPolicy` y `defaultWarehouse`. |
| `clientPortal` | `includeLinkInEmails`. |
| `inbox` | Canales de entrada de la bandeja: `email` (`active`, `address`, `autoScan`) y `googleDrive` (`linked`, `folderId`, `folderUrl`, `error`). Detalle en [Bandeja de entrada](./inbox.md#cómo-llegan-documentos-a-la-bandeja). |
| `modules` | Funcionalidades disponibles según el plan: `accounting`, `inventory`, `stockControl`, `inbox`, `googleDrive`, `payrollScanning`, `multicurrency`, `clientPortal`, `recurringInvoices` y `scans` (`period` y `limit`, con `-1` ilimitados). |
| `webUrl` | Página de Ajustes en la aplicación web, para que un usuario cambie esta configuración. |

**Notas:**

- **Es solo lectura.** Activar el buzón, enlazar Google Drive, cambiar el
  modo borrador o dar de alta VeriFactu se hace desde la interfaz
  (`webUrl`).
- **`inbox.email.address` es confidencial.** Cualquier correo que llegue a
  esa dirección entra en la bandeja de la empresa sin verificar el
  remitente. Dásela al usuario de la empresa, no la publiques.
- **`inbox.email.autoScan`** indica si lo que llega por correo se escanea
  al momento. Si es `false`, los documentos quedan en la bandeja con
  `status: pending` hasta que un usuario los escanee.
- **`inbox.googleDrive.error`** distinto de `null` significa que la
  sincronización está parada (credenciales caducadas o carpeta borrada) y
  el usuario tiene que volver a enlazar la carpeta.
- **`company.sandbox`** marca los entornos de pruebas: sus documentos no
  tienen validez y no se envían a la AEAT ni a las diputaciones.
- Lo que `modules` marca como no disponible devuelve `403` o `409` en los
  endpoints que lo requieren.
- Esta configuración cambia con muy poca frecuencia: cachéala por empresa
  y refréscala al arrancar una sesión o si una llamada falla por un motivo
  de configuración.

###### Ejemplo de respuesta

```json
{
  "content": {
    "company": {
      "id": "com_4d8f0a6e-1b2c-4c3d-9e8f-0a1b2c3d4e5f",
      "sandbox": false,
      "name": "Cafés Zamora SL",
      "surname": null,
      "brand": "Cafés Zamora",
      "taxCode": "B12345674",
      "email": "hola@cafeszamora.example",
      "phone": "980 000 000",
      "website": "https://cafeszamora.example",
      "address": { "details": "Pza Mayor, 4", "zipcode": "49004", "city": "Zamora", "province": "Zamora", "country": "ES" },
      "country": "ES",
      "businessType": "EM",
      "taxRegion": "ES_ES",
      "currency": "EUR",
      "creationDate": "2024-03-12T09:15:00.000Z"
    },
    "locale": { "timezone": "Europe/Madrid", "language": "es", "dateFormat": "DD-MM-YYYY", "timeFormat": "HH:mm", "dateTimeFormat": "DD-MM-YYYY HH:mm" },
    "sales": { "newInvoiceMode": "final", "taxCalculationMode": "total", "externalInvoicesEnabled": false },
    "expenses": { "newBillMode": "final", "taskToExpenseDefaultDate": "invoiceDateOrEmpty" },
    "taxes": { "defaultSales": "BC", "equivalenceSurcharge": false, "applyVatEUToFinalConsumer": false, "vatEU": "ESB12345674" },
    "numbering": { "zeros": 4, "position": "prefix" },
    "accounting": { "startDate": "2024-01-01", "lockDate": "2025-12-31" },
    "fiscal": {
      "verifactu": { "enabled": true, "environment": "verifactu_production", "mode": "mode_verifactu", "startDate": "2026-01-01", "endDate": null, "exemption": null },
      "ticketbai": { "enabled": false, "environment": null, "region": null, "startDate": null }
    },
    "email": { "from": ["facturas@cafeszamora.example"] },
    "clientPortal": { "includeLinkInEmails": true },
    "inbox": {
      "email": { "active": true, "address": "cafes_zamora-a1b2c3d4e5@inbox.facturadirecta.com", "autoScan": true },
      "googleDrive": { "linked": true, "folderId": "1AbCdEfGhIjKlMnOpQrStUvWxYz", "folderUrl": "https://drive.google.com/drive/folders/1AbCdEfGhIjKlMnOpQrStUvWxYz", "error": null }
    },
    "modules": {
      "accounting": true, "inventory": false, "stockControl": false, "inbox": true, "googleDrive": true,
      "payrollScanning": true, "multicurrency": false, "clientPortal": true, "recurringInvoices": true,
      "scans": { "period": "total", "limit": -1 }
    },
    "webUrl": "https://3.facturadirecta.com/company/com_4d8f0a6e-1b2c-4c3d-9e8f-0a1b2c3d4e5f/settings"
  }
}
```

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/settings/company"
```

## Lista de series de numeración

`GET /{companyId}/settings/series/{documentType}` devuelve las series
de numeración disponibles para un tipo de documento.

**Parámetro de path:**

- **`documentType`** (obligatorio) — uno de:
  - `invoice` — facturas de venta.
  - `estimate` — presupuestos.
  - `deliveryNote` — albaranes.
  - `clientOrder` — pedidos de cliente.
  - `purchaseOrder` — órdenes de compra.

**Parámetros globales aceptados:** `accept-version`.

**Respuesta:** `{ items: SeriesItem[] }` sin paginar.

Cada `SeriesItem` tiene:

| Campo | Tipo | Significado |
|---|---|---|
| `serie` | string | Valor de la serie. Puede contener `##` o `####`, que se sustituye por el año de la fecha del documento en el momento de guardar. |
| `reset` | `"never"` \| `"year"` | Si reinicia la numeración con el cambio de año (`year`) o si continúa desde el último número del año anterior (`never`). |
| `manual` | boolean | Si la serie acepta numeración manual desde la interfaz. |
| `notes` | string | Notas descriptivas. |
| `theme` | string | ID efectivo de la plantilla de la serie. Si la serie no tiene una asociación específica, devuelve la predeterminada de la empresa. |
| `invoiceType` | enum | Solo en series de facturas. Valores: `complete`, `complete_correction`, `simplified`, `simplified_correction`, `external`. |
| `correction` | boolean | **OBSOLETO** — usar `invoiceType` en su lugar. Indica si la serie es para rectificativas. |

**Notas:**

- Las series se configuran desde la interfaz; este endpoint solo las
  expone. La API pública no crea, modifica ni borra series.
- Las series de pedidos de cliente y de órdenes de compra solo existen si la
  empresa las ha configurado; si no, `items` es una lista vacía.
- Para series de facturas, `invoiceType` distingue los regímenes
  fiscales aplicables. El campo `correction` queda por
  compatibilidad pero **no debe usarse en código nuevo**.
- `##`/`####` en `serie` se sustituye por el año al guardar el
  documento. Ejemplo: serie `"F-##-"` con fecha 2026 produce números
  como `F-26-1`, `F-26-2`...
- Si una serie no tiene plantilla propia, `theme` contiene la predeterminada
  vigente de la empresa. No presupongas que siempre será `thm_master`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/settings/series/invoice"
```

## Lista de plantillas

`GET /{companyId}/settings/themes` devuelve las plantillas de
impresión configuradas en la empresa.

**Parámetros globales aceptados:** `accept-version`.

**Respuesta:** `{ items: ThemeItem[] }` sin paginar.

Cada `ThemeItem` tiene:

| Campo | Tipo | Significado |
|---|---|---|
| `id` | string | Identificador de la plantilla. Es lo que se usa en `content.main.theme` de los documentos. |
| `title` | string | Nombre legible. |

**Notas:**

- Las plantillas se configuran desde la interfaz; este endpoint solo
  las expone.
- Para aplicar una plantilla concreta a un documento, indica su `id` en
  `content.main.theme` al crear o actualizar el documento.
- Si omites `content.main.theme` al crear un documento, FacturaDirecta usa la
  plantilla de su serie y, si la serie no tiene una asociación específica, la
  predeterminada de la empresa. `thm_master` es la alternativa final cuando no
  hay una selección válida.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/settings/themes"
```

## Recomendaciones

- **Empieza por `/settings/company`** al abrir una sesión con una empresa:
  te dice en qué modo trabaja, qué módulos tiene y por dónde puede recibir
  documentos, y evita llamadas que fallarían por configuración.
- **Cachea el catálogo** localmente por empresa. La ficha, las series, los
  themes y los impuestos cambian con muy baja frecuencia; consultarlos en
  cada llamada gasta cuota innecesariamente.
- **Refresca al fallar**: si una creación falla con `400` por
  `theme` o `serie` desconocido, refresca el catálogo: alguien puede
  haber cambiado la configuración desde la interfaz.

## Referencia exhaustiva

Esta página cubre los matices funcionales de `company`, `series` y
`themes`.
Para impuestos, la referencia funcional está en la
[guía de Impuestos](../guides/taxes.md). Para el contrato completo
campo a campo, consulta el
[Swagger UI](https://www.facturadirecta.com/api) o el
[openapi crudo](https://app.facturadirecta.com/openapi.json).

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| GET | `/{companyId}/settings/company` | `getCompanySettings` | `settings:read` | Ficha y forma de trabajar de la empresa |
| GET | `/{companyId}/settings/series/{documentType}` | `getSeries` | `settings:read` | Lista de series de numeración |
| GET | `/{companyId}/settings/taxes/purchases` | `getPurchasesTaxes` | `settings:read` | Lista de impuestos de compra disponibles |
| GET | `/{companyId}/settings/taxes/sales` | `getSalesTaxes` | `settings:read` | Lista de impuestos de venta disponibles |
| GET | `/{companyId}/settings/themes` | `getThemes` | `settings:read` | Lista de plantillas de documentos disponibles |

## Scopes

- **`settings:read`** — Lectura de ajustes/datos de configuración.
