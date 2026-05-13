---
title: Uploads (subida de archivos)
audience:
  - developers
status: draft
---

# Uploads (subida de archivos)

El recurso `uploads` permite **subir archivos binarios a FacturaDirecta**
para luego vincularlos como adjuntos a documentos (facturas, gastos,
presupuestos, albaranes, nóminas, facturas recurrentes) o como
contenido de un [item de la bandeja de entrada](./inbox.md).

Un upload es un recurso **temporal**: tiene TTL de 24 horas. Si no se
vincula a ningún documento en ese plazo, el sistema lo purga
automáticamente.

> En los ejemplos, `upl_…` son ilustrativos. El ID real se devuelve al
> crear cada upload.

## Dos modos de subida

`POST /uploads` admite dos modos según el `Content-Type` del request:

| Modo | Content-Type | Tamaño máximo | Cuándo usar |
|---|---|---|---|
| **Multipart** | `multipart/form-data` | **10 MB** | Por defecto. Subes el binario en una sola petición y recibes un upload `committed` listo para vincular. Es el modo recomendado para clientes y agentes. |
| **Presigned** | `application/json` | **25 MB** | Cuando el archivo es grande o quieres evitar pasar el binario por la API. Recibes una URL pre-firmada de S3 y subes el binario directo allí; luego sellas el upload con `POST /uploads/{uploadId}/commit`. |

Ambos modos requieren el scope **`files:write`**.

Si en multipart se excede el límite, la API devuelve `413` con
`code: "upload_too_large_for_multipart"` y un `hint` apuntando al modo
presigned.

## Detección automática de Facturae

Al sellar el upload (en multipart sucede automáticamente; en presigned
ocurre al llamar al endpoint `commit`), el sistema **detecta si el
archivo es un Facturae** válido. Si lo es, el campo
`content.contentSubtype` del upload tiene valor `"facturae"`. Cuando se
vincule a una factura de compra vía la bandeja de entrada, el sistema
usa un extractor nativo en vez de LLM (exacto y rápido).

## Operaciones

- [Crear upload (subida)](#crear-upload-subida)
- [Sellar upload presigned (commit)](#sellar-upload-presigned-commit)

## Crear upload (subida)

`POST /{companyId}/uploads` sube un archivo. Selecciona el modo según
el `Content-Type` del request.

**Parámetros globales aceptados:** `accept-version`.

### Modo multipart (recomendado, hasta 10 MB)

Envía el binario en el campo `file` de un `multipart/form-data`.

**Respuesta:** `{ content: PublicApiUpload }` con el upload ya en
estado `committed`, listo para vincular.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  -F "file=@./factura.pdf" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/uploads"
```

Respuesta típica:

```json
{
  "content": {
    "id": "upl_4a8c2e91-3b5d-4f7c-9e1a-2d4b6c8e0a1f",
    "name": "factura.pdf",
    "size": 245678,
    "contentType": "application/pdf",
    "contentSubtype": null,
    "expiresAt": "2026-05-14T10:22:31.000Z"
  }
}
```

### Modo presigned (hasta 25 MB)

Para archivos grandes o cuando prefieres no pasar el binario por la
API.

**Parámetros del body** (`application/json`):

- **`filename`** (obligatorio) — nombre del archivo (incluyendo
  extensión).
- **`contentType`** (opcional) — MIME del archivo. Si no se indica, se
  usa `application/octet-stream`.

**Respuesta:** `{ content: PublicApiPresignedUpload }` con:

- `id` — el `upl_<uuid>` del upload (aún no `committed`).
- `signedRequest` — URL pre-firmada de S3 para hacer `PUT` con el
  binario.
- `expiresAt` — fecha límite para completar PUT + commit.

Tras subir a S3, sella el upload con
[`POST /uploads/{uploadId}/commit`](#sellar-upload-presigned-commit).

###### Ejemplo de request JSON

```json
{
  "filename": "factura.pdf",
  "contentType": "application/pdf"
}
```

###### Copy as cURL (presigned, flujo completo)

```shell
# 1. Solicitar URL presigned
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"filename":"factura.pdf","contentType":"application/pdf"}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/uploads"

# Respuesta: { content: { id: "upl_...", signedRequest: "https://...s3...", ... } }

# 2. PUT del binario a la URL pre-firmada
curl -s -X PUT \
  -H "Content-Type: application/pdf" \
  --data-binary @./factura.pdf \
  "<signedRequest>"

# 3. Sellar el upload (commit)
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  -X POST \
  "https://app.facturadirecta.com/api/$COMPANY_ID/uploads/upl_4a8c2e91-3b5d-4f7c-9e1a-2d4b6c8e0a1f/commit"
```

## Sellar upload presigned (commit)

`POST /{companyId}/uploads/{uploadId}/commit` cierra un upload creado
en modo presigned. El endpoint:

1. Verifica que el binario está en S3.
2. Comprueba que el tamaño no excede 25 MB. Si lo excede, devuelve
   `413` con `code: "upload_too_large_for_presigned"` y el upload pasa
   a estado `expired`.
3. Detecta si el archivo es un Facturae y, en su caso, marca
   `contentSubtype: "facturae"`.
4. Deja el upload en estado `committed`, listo para vincular a un
   documento.

**Sin body.** Solo el `uploadId` en el path.

**Respuesta:** `{ content: PublicApiUpload }` con los datos del upload
ya sellado.

**Parámetros globales aceptados:** `accept-version`.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -X POST \
  "https://app.facturadirecta.com/api/$COMPANY_ID/uploads/upl_4a8c2e91-3b5d-4f7c-9e1a-2d4b6c8e0a1f/commit"
```

## Vincular un upload

Una vez `committed`, el upload se vincula a un documento o item de la
bandeja desde la API del recurso correspondiente. Ejemplos:

- **A una factura de compra:**
  `POST /bills/{id}/attachments` con `{ "uploadIds": ["upl_..."] }`.
  Ver [Adjuntos en bills](./bills.md#adjuntos).
- **A un presupuesto, albarán, nómina, factura recurrente:** misma
  forma en sus respectivos endpoints `/attachments`.
- **A la bandeja de entrada:**
  `POST /inbox` con `{ "upload": "upl_..." }`. El upload se convierte
  en un inbox item; el sistema lo escanea para extraer datos. Ver
  [Crear item](./inbox.md#crear-item-subir-documento).

Tras vincular, el upload **deja de ser temporal** y persiste como
parte del adjunto del documento.

## Recomendaciones

- **Usa multipart por defecto.** Es más simple y cubre la mayoría de
  PDFs de facturas y nóminas (típicamente < 10 MB).
- **Cambia a presigned para archivos grandes** o cuando tu pipeline
  ya hace uploads directos a S3 (evita doble tráfico de red).
- **Vincula pronto.** Los uploads tienen TTL de 24 h; un upload sin
  vincular se purga al pasar ese plazo.
- **Para integraciones inbox**, el flujo natural es: `POST /uploads`
  (multipart) → `POST /inbox` (con el `upl_*` resultante). El sistema
  escanea automáticamente.
- **Detecta `contentSubtype: "facturae"` en la respuesta** para
  identificar Facturae sin parsear el XML tú mismo.

## Errores comunes

- `413 Payload Too Large` (`code: "upload_too_large_for_multipart"`) —
  archivo > 10 MB en modo multipart. Cambia a modo presigned.
- `413 Payload Too Large` (`code: "upload_too_large_for_presigned"`) —
  archivo > 25 MB tras el commit. El upload pasa a `expired`. Para
  archivos mayores, no hay forma de subirlos por la API pública.
- `400 ValidationError` — `filename` ausente en modo presigned.
- `404 Not Found` — `uploadId` no existe o pertenece a otra empresa.
- `409 Conflict` — `commit` sobre un upload que ya está `committed`
  o `expired`.

Ver [Errores y validaciones](../guides/errors.md) para el formato
general.

## Referencia exhaustiva

Esta página cubre los dos modos y el flujo completo. Para el contrato
exhaustivo del schema, consulta el
[Swagger UI](https://www.facturadirecta.com/api) o el
[openapi crudo](https://app.facturadirecta.com/openapi.json).

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| POST | `/{companyId}/uploads` | `createUpload` | `files:write` | Subir un archivo |
| POST | `/{companyId}/uploads/{uploadId}/commit` | `commitUpload` | `files:write` | Sellar un upload presigned tras subirlo a S3 |

## Scopes

- **`files:write`** — Subida de archivos para adjuntar a documentos.
