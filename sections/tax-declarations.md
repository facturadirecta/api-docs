---
title: Declaraciones de impuestos
audience:
  - developers
status: draft
---

# Declaraciones de impuestos

Una **declaración de impuestos** (`taxDeclaration`) es un **modelo de
impuestos** que el usuario ha generado y guardado en FacturaDirecta desde
la sección **Impuestos**: un 303 trimestral, un 347 anual, un 130… Es la
misma información que ve el usuario en la web: el número de modelo, el
periodo, el resultado, las casillas que se escriben en el fichero de
presentación de la AEAT y, en las declaraciones informativas, los
terceros declarados.

El recurso es de **solo lectura**. La API no genera, rellena ni
presenta modelos: el usuario los genera en la web y los presenta en la
Sede de la AEAT. Lo que sí hace la API es dejar **descargar el PDF y el
fichero de presentación** de un modelo ya guardado, los mismos que
descargan los botones de la web.

No confundas este recurso con el **catálogo de impuestos** de la empresa
(los tipos de IVA, IRPF o IGIC que se aplican a las líneas de los
documentos), que está en `/settings/taxes/*` y se describe en la
[guía del catálogo de impuestos](../guides/taxes.md).

## Permisos

- El scope es `taxDeclarations:read`. Con OAuth, el rol del usuario en
  la empresa necesita además el permiso de **impuestos** en lectura, el
  mismo que da acceso a la sección Impuestos de la web.
- Una API key accede si tiene el scope. Al crearla, solo puede concederle
  `taxDeclarations:read` quien tiene el permiso de impuestos.
- El IBAN de domiciliación o devolución que algunos modelos llevan en sus
  casillas sale **enmascarado** (`****1332`) salvo que el token o la API
  key tengan también `banks:readIban`, igual que en
  [bancos](./banks.md) y [métodos de pago](./payment-methods.md).
- Leer los modelos no depende de los módulos del plan: si el usuario pudo
  generarlos, la API los devuelve.

## Dos capas: el sobre y las casillas

Cada declaración tiene dos partes con compromisos de estabilidad
distintos:

- **El sobre** es todo lo que no son casillas: identificador, modelo,
  periodo, resultado, estado, origen, fichero de presentación disponible
  y número de adjuntos. Es el contrato de la API: sus campos no cambian
  de significado y solo se añaden.
- **Las casillas** (`boxes`) y los **registros por tercero** (`records`)
  siguen el **diseño de registro de la AEAT** que indica `formVersion`.
  Para una `formVersion` dada el conjunto de claves no cambia nunca; cuando
  la AEAT publica un diseño nuevo, los modelos generados a partir de
  entonces llevan otra `formVersion` y pueden traer casillas nuevas o
  dejar de traer alguna. Trata las claves como un conjunto abierto.

Los valores de `form`, `kind`, `status`, `origin` y
`result.declarationType` son **enums abiertos**: tu integración debe
tolerar valores que no conozca.

## Estructura de una declaración

```json
{
  "id": "tax_7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "form": "303",
  "kind": "selfAssessment",
  "fiscalYear": 2026,
  "fiscalPeriod": "3T",
  "periodStart": "2026-07-01",
  "periodEnd": "2026-09-30",
  "generatedAt": "2026-10-14T09:12:33.000Z",
  "formVersion": "303_e26v101",
  "result": { "amount": 1842.5, "currency": "EUR", "declarationType": "U" },
  "status": "generated",
  "filing": null,
  "origin": "facturadirecta",
  "aeatFile": { "available": true, "filename": "modelo_303.303" },
  "attachmentsCount": 1,
  "boxes": [
    { "key": "ejercicio", "title": "Ejercicio devengo", "value": "2026" },
    { "key": "periodo", "title": "Período.", "value": "3T" },
    { "key": "tipoDeclaracion", "title": "Tipo Declaración", "value": "U" },
    { "key": "07", "title": "Liquidación (3) - Regimen General - IVA Devengado - Base imponible 21%", "value": 12500 },
    { "key": "09", "title": "Liquidación (3) - Regimen General - IVA Devengado - Cuota 21%", "value": 2625 },
    { "key": "27", "title": "Liquidación (3) - Regimen General - IVA Devengado - Total cuota devengada ( [152] + [167] + [03] + [155] + [06] + [09] + [11] + [13] + [15] + [158] + [170] + [18] + [21] + [24] + [26] )", "value": 2625 },
    { "key": "45", "title": "Liquidación (3) - Regimen General - IVA Deducible - Total a deducir ( [29] + [31] + [33] + [35] + [37] + [39] + [41] + [42] + [43] + [44] ) - Cuota", "value": 782.5 },
    { "key": "71", "title": "Resultado - Resultado ( [69] - [70] + [109] - [112] )", "value": 1842.5 },
    { "key": "iban", "title": "Domiciliación/Devolución - IBAN", "value": "****1332" }
  ]
}
```

El ejemplo está recortado: un 303 real trae unas sesenta casillas, y en
el listado solo aparecen con `related=boxes`.

| Campo | Tipo | Significado |
|---|---|---|
| `id` | string | Identificador opaco y estable del modelo (`tax_` y un uuid v4). Sirve en el resto de operaciones y en [adjuntos](#listar-los-adjuntos-de-una-declaración) y [actividad](./activity.md). |
| `form` | string | Número del modelo de la AEAT: `303`, `111`, `115`, `130`, `202`, `180`, `190`, `347`, `349` o `390`. Enum abierto. |
| `kind` | string | `selfAssessment` en las autoliquidaciones (303, 111, 115, 130, 202), que liquidan un importe; `informative` en las declaraciones informativas (180, 190, 347, 349, 390), que solo declaran operaciones o terceros. |
| `fiscalYear` | entero | Ejercicio. |
| `fiscalPeriod` | string \| null | Trimestre (`1T`…`4T`), mes (`01`…`12`) o pago fraccionado del 202 (`1P`…`3P`). `null` en los modelos anuales. |
| `periodStart`, `periodEnd` | date | Primer y último día del periodo declarado. En los anuales y en el 202, el ejercicio entero. |
| `generatedAt` | date-time \| null | Momento en que se calcularon las casillas. Si el usuario regenera el modelo, cambia. |
| `formVersion` | string | Diseño de registro de la AEAT aplicado, por ejemplo `303_e26v101`. Ver [las versiones por modelo](#versiones-del-diseño-de-registro). |
| `result` | objeto | `amount`: resultado de la autoliquidación en `currency` (positivo a ingresar, negativo a compensar o devolver; `null` en las informativas). `declarationType`: código de la AEAT del tipo de declaración, cuando el modelo lo tiene. |
| `status` | string | Hoy siempre `generated`: FacturaDirecta no registra la presentación. Enum abierto: llegarán otros valores (como `filed`) sin cambio de versión de la API. |
| `filing` | objeto \| null | Datos de presentación (fecha, CSV, NRC, canal). Hoy nada los registra y vale siempre `null`; el campo existe para que el contrato no cambie cuando se registren. |
| `origin` | string | Dónde se generó el modelo. Hoy siempre `facturadirecta`. Enum abierto. |
| `aeatFile` | objeto | `available` indica si el modelo tiene fichero de presentación (el 390 no: se rellena en la Sede) y `filename` el nombre con el que lo devuelve la API. |
| `attachmentsCount` | entero | Adjuntos del modelo, por ejemplo el justificante de presentación si el usuario lo adjuntó. |
| `boxes` | array | Casillas del modelo (ver abajo). Siempre en el detalle; en el listado, con `related=boxes`. |
| `records` | objeto \| null | Registros por tercero de las informativas (ver abajo). Solo en el detalle, con `related=records`. |

### Los códigos de `result.declarationType`

Son los del diseño de registro de la AEAT y solo existen en las
autoliquidaciones que lo llevan (303, 130…): `I` ingreso, `U`
domiciliación del ingreso, `G` cuenta corriente tributaria (ingreso),
`C` solicitud de compensación, `D` devolución, `V` cuenta corriente
tributaria (devolución) y `N` sin actividad o resultado cero.

### Las casillas: `boxes`

`boxes` son las casillas **tal como se escriben en el fichero de
presentación de la AEAT**, en el orden del diseño de registro, también
las que valen cero. Cada una lleva:

- **`key`** — el número oficial de la casilla (`27`, `71`) o la clave de
  control del diseño de registro: `ejercicio`, `periodo`,
  `tipoDeclaracion`, `iban`, `bic`, las marcas de régimen del 303
  (`RECC`, `prorrata`, `volOperaciones`…) o los totales de cabecera de
  las informativas (`numPersonas`, `importeAnual`).
- **`title`** — la descripción de la casilla en el formulario. Es texto
  de ayuda, no contrato.
- **`value`** — un número en las casillas de importe y de contadores; un
  código o texto en las demás.

No se publican el relleno técnico del fichero (constantes, longitudes,
sello), la identidad de la empresa (NIF y razón social, que ya da
[`GET /settings/company`](./settings.md)) ni los registros por tercero,
que van en `records`. Para interpretar una casilla concreta, usa las
instrucciones del modelo de la AEAT para la `formVersion` indicada.

### Los terceros: `records`

Las declaraciones informativas no liquidan nada: declaran con quién se
ha operado, y su fichero lleva un registro por cada tercero. `records`
los devuelve con dos listas: `fields`, los campos de cada registro
(`key` y `title`, en el orden del diseño), e `items`, un objeto por
tercero con el valor de cada campo bajo su `key`.

| Modelo | Registros | Campos típicos |
|---|---|---|
| 347 | Clientes y proveedores con más de 3.005,06 € en el año | `fiscalId`, `name`, `provincia`, `pais`, `clave`, `importeAnual`, `importe1T`…`importe4T` |
| 349 | Operadores intracomunitarios | `fiscalId`, `name`, `clave`, `base` |
| 190 | Perceptores con retención (empleados, profesionales) | `fiscalId`, `name`, `region`, `clavePercepcion`, `subclavePercepcion`, percepciones y retenciones |
| 180 | Arrendadores de inmuebles urbanos | `fiscalId`, `name`, `region`, `modalidad`, `baseRetencion`, `porcentaje`, `retencion` |

Los campos exactos de cada modelo son los de su `formVersion`. Los
registros llevan datos personales de terceros (NIF y nombre), los mismos
que ya puedes leer con `contacts:read` y `bills:read`: trátalos con el
mismo cuidado.

### Versiones del diseño de registro

`formVersion` identifica el diseño de registro de la AEAT con el que se
generó el modelo y, por tanto, qué casillas tiene. Se elige por la fecha
del modelo (el último día de su periodo) y se queda fijo en el documento
guardado. Los valores actuales, del más reciente al más antiguo:

| Modelo | `formVersion` |
|---|---|
| 303 | `303_e26v101` (desde 2026), `303_e25v102` (2025), `303_e24v201` (desde el 4T de 2024), `303_e24v201_3T` (3T de 2024), `303_e23`, `303_e22`, `303_e21v200` (desde el 3T de 2021), `303_e21v103`, `303_e19v10_10`, `303_e18v10_20`, `303_e18v10_10`, `303_e17v20_04`, `303_e15v32` |
| 390 | `390_2026`, `390_2025`, `390_2024`, `390_2023`, `390_2022`, `390_2021`, `390_2018`, `390_2017` |
| 190 | `190_2025`, `190_2024`, `190_2023`, `190_2022`, `190` |
| 347 | `347_e25` (desde 2025), `347` |
| 202 | `202_e25` (desde 2025), `202_e23` (desde 2023), `202_e17v48` |
| 130 | `130_e15v12` (desde el 2T de 2024), `130_e15v11` |
| 111, 115, 180, 349 | `111`, `115`, `180`, `349` |

<!-- review: tabla derivada de TAX_DEFINITION_VERSIONS en packages/model/src/model/taxes/definitions/index.ts; verificado 2026-10-26 -->

## Operaciones

- [Listar declaraciones](#listar-declaraciones)
- [Obtener una declaración](#obtener-una-declaración)
- [Generar el PDF de una declaración](#generar-el-pdf-de-una-declaración)
- [Generar el fichero de presentación](#generar-el-fichero-de-presentación)
- [Listar los adjuntos de una declaración](#listar-los-adjuntos-de-una-declaración)

## Listar declaraciones

`GET /{companyId}/taxDeclarations` devuelve los modelos guardados de la
empresa con **paginación por offset** (patrón A de la
[guía de paginación](../guides/pagination.md)). Los modelos eliminados no
salen. Sin ordenación explícita, van del más reciente al más antiguo por
la fecha del modelo.

**Respuesta:** `{ pagination, items: TaxDeclarationResponse[] }`. Cada
elemento lleva `content` (la declaración, sin `boxes` salvo que los
pidas), `tags`, `creationDate`, `modificationDate`, `webUrl` (la página
del modelo en la web) y, con `related=journal`, `related.journal`.

**Parámetros de consulta:**

- `form` — número de modelo. Se repite para indicar varios
  (`form=303&form=390`). Un valor que no es un modelo conocido responde
  `400`.
- `kind` — `selfAssessment` o `informative`. Se puede combinar con
  `form`; se devuelven los modelos que cumplen los dos.
- `fiscalYear` — ejercicio.
- `fiscalPeriod` — `1T`…`4T`, `01`…`12` o `1P`…`3P`. Admite varios. Los
  modelos anuales no tienen periodo y no salen al filtrar por él.
- `minDate` / `maxDate` — fecha mínima y máxima del modelo, ambas
  incluidas, en formato `YYYY-MM-DD`. La fecha de un modelo es el último
  día de su periodo (el 30 de septiembre para un 3T, el 31 de diciembre
  para un anual).
- `status` — hoy solo admite `generated`, que es el estado de todos los
  modelos guardados; el filtro existe para que puedas fijarlo desde el
  principio y no restringe nada.
- `sortBy` — `date`, `total` (el resultado), `fiscalYear`,
  `creationDate` o `modificationDate`, con `-` delante para el orden
  descendente. Admite varios: `sortBy=-fiscalYear&sortBy=date`. A igual
  criterio, los resultados se ordenan por fecha de creación descendente.
- `related` — `boxes` añade las casillas a cada elemento; `journal`, su
  asiento contable en `related.journal`. Los registros por tercero solo
  se piden en el detalle: `related=records` responde `400` aquí.

**Parámetros globales aceptados:**

- Etiquetas: `allTheseTags`, `anyOfTheseTags`, `hasTags`.
- Fechas de alta y modificación: `minCreationDate`, `maxCreationDate`,
  `minModificationDate`, `maxModificationDate`.
- Paginación: `limit` (por defecto `25`, máximo `500`) y `offset`.
- El header `accept-version`.

Ver la [guía de paginación](../guides/pagination.md) y
[Autenticación](../guides/authentication.md) para el detalle.

###### Copy as cURL

Modelos de IVA de 2026 con sus casillas:

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/taxDeclarations?form=303&fiscalYear=2026&related=boxes"
```

Declaraciones informativas de 2025:

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/taxDeclarations?kind=informative&fiscalYear=2025"
```

## Obtener una declaración

`GET /{companyId}/taxDeclarations/{id}` devuelve un modelo con sus
**casillas** completas.

**Parámetros de consulta:**

- `related` — `records` añade los registros por tercero (solo tienen las
  informativas; en las demás `records` vale `null`) y `journal` el asiento
  contable del modelo en `related.journal`.

**Parámetros globales aceptados:** el header `accept-version`.

**Respuesta:** `{ content: TaxDeclaration, tags, creationDate, modificationDate, webUrl, related? }`.

Un modelo eliminado responde `404`.

###### Ejemplo de respuesta

Un 347 con sus declarados:

```json
{
  "content": {
    "id": "tax_0b6f0f0e-1f3c-4a2d-9c4e-7a8b9c0d1e2f",
    "form": "347",
    "kind": "informative",
    "fiscalYear": 2025,
    "fiscalPeriod": null,
    "periodStart": "2025-01-01",
    "periodEnd": "2025-12-31",
    "generatedAt": "2026-01-20T08:00:00.000Z",
    "formVersion": "347_e25",
    "result": { "amount": null, "currency": "EUR", "declarationType": null },
    "status": "generated",
    "filing": null,
    "origin": "facturadirecta",
    "aeatFile": { "available": true, "filename": "modelo_347.347" },
    "attachmentsCount": 0,
    "boxes": [
      { "key": "ejercicio", "title": "Ejercicio", "value": "2025" },
      { "key": "contactPhone", "title": "Teléfono", "value": 600000000 },
      { "key": "contactName", "title": "Apellidos y nombre de la persona con quien relacionarse", "value": "PEPA PEREZ" },
      { "key": "numPersonas", "title": "Número total de personas y entidades (número de registros tipo 2)", "value": 2 },
      { "key": "importeAnual", "title": "Importe total anual de las operaciones", "value": 9000 }
    ],
    "records": {
      "fields": [
        { "key": "fiscalId", "title": "NIF del declarado" },
        { "key": "name", "title": "Apellidos y nombre, razon social o denominación del declarado" },
        { "key": "provincia", "title": "Código provincia" },
        { "key": "pais", "title": "Código pais" },
        { "key": "clave", "title": "Clave de operación (A..G)" },
        { "key": "importeAnual", "title": "Importe anual de las operaciones" }
      ],
      "items": [
        { "fiscalId": "12345678Z", "name": "CLIENTE UNO", "provincia": "08", "pais": "ES", "clave": "B", "importeAnual": 5000 },
        { "fiscalId": "87654321X", "name": "PROVEEDOR DOS", "provincia": "28", "pais": "ES", "clave": "A", "importeAnual": 4000 }
      ]
    }
  },
  "tags": [],
  "creationDate": "2026-01-20T08:00:05.000Z",
  "modificationDate": "2026-01-20T08:00:05.000Z",
  "webUrl": "https://3.facturadirecta.com/company/<id-empresa>/doc/tax_0b6f0f0e-1f3c-4a2d-9c4e-7a8b9c0d1e2f"
}
```

`<id-empresa>` es un placeholder. Las casillas y los campos de los
registros están recortados.

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/taxDeclarations/tax_0b6f0f0e-1f3c-4a2d-9c4e-7a8b9c0d1e2f?related=records"
```

## Generar el PDF de una declaración

`PUT /{companyId}/taxDeclarations/{id}/pdf` genera el PDF resumen del
modelo, con sus casillas y valores, el mismo que imprime la web, y
devuelve una URL temporal desde la que descargarlo. Funciona igual que el
PDF de las [facturas](./invoices.md#generar-pdf): el cuerpo lleva
`mode` (`attachment`, `inline` o `print`) y la respuesta `url`,
`filename` (`modelo_303_2026_3T.pdf`), `availableUntil` y, en modo
`print`, `urlPdf`.

El PDF se genera solo si aún no existe o si el modelo ha cambiado desde
la última generación; si no, se reutiliza el ya generado, como el PDF de
las facturas. No calcula ni modifica el modelo, y cada generación (no
cada descarga) deja en la [actividad](./activity.md) del documento la
misma traza `file` (formato `pdf`) que la web. No admite la cabecera
`Idempotency-Key`: repetir la llamada devuelve el mismo fichero.

###### Copy as cURL

```shell
curl -s -X PUT -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d '{"mode": "attachment"}' \
  "https://app.facturadirecta.com/api/$COMPANY_ID/taxDeclarations/tax_7c9e6679-7425-40de-944b-e07fc1f90ae7/pdf"
```

## Generar el fichero de presentación

`PUT /{companyId}/taxDeclarations/{id}/aeatFile` devuelve el **fichero de
presentación de la AEAT** (el diseño de registro BOE) de un modelo ya
guardado, el mismo que descarga el botón «Descargar fichero AEAT» de la
web, como `{ url, filename }`. `filename` es el que indica
`aeatFile.filename` en la declaración (`modelo_303.303`).

El fichero se compone solo si aún no existe o si el modelo ha cambiado
desde la última generación; si no, se devuelve el ya generado, que
normalmente es el que la web generó al crear el modelo. Funciona como el
[Facturae de las facturas](./invoices.md#generar-facturae). Se construye
a partir de las casillas guardadas: no calcula ni modifica nada, y cada
generación (no cada descarga) deja en la [actividad](./activity.md) la
misma traza `file` (formato `boefile`) que la web. Presentarlo sigue
siendo cosa del usuario, en la Sede de la AEAT. No admite la cabecera
`Idempotency-Key`: repetir la llamada devuelve el mismo fichero.

El 390 no tiene fichero (se rellena en la Sede) y responde `404`;
compruébalo antes con `aeatFile.available`.

###### Copy as cURL

```shell
curl -s -X PUT -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/taxDeclarations/tax_7c9e6679-7425-40de-944b-e07fc1f90ae7/aeatFile"
```

## Listar los adjuntos de una declaración

`GET /{companyId}/taxDeclarations/{id}/attachments` devuelve
`{ items: Attachment[] }` con los ficheros que el usuario adjuntó al
modelo desde la web, por ejemplo el justificante de presentación. La API
no interpreta su contenido. Cada adjunto lleva `name`, `url`,
`creationTime` y, cuando se conoce, `size`.

Añadir o borrar adjuntos por la API no está disponible en este recurso.

## Recomendaciones

- **Para saber si una empresa ha generado un modelo**, filtra por `form`,
  `fiscalYear` y `fiscalPeriod`. Un modelo que no aparece no se ha
  generado en FacturaDirecta; uno que aparece se ha generado, pero la API
  no sabe si se ha presentado (`status` es siempre `generated` y `filing`
  es `null`).
- **Guarda `formVersion` junto a las casillas** que proceses. Es lo que te
  dice qué significa cada `key` y te avisa de que un modelo nuevo puede
  traer casillas distintas.
- **Pide `related=boxes` solo cuando las necesites**: un 303 trae unas
  sesenta casillas y un listado con ellas crece rápido. Para los terceros
  de una informativa, usa el detalle con `related=records`.
- **Compara `generatedAt`** si guardas copias: si el usuario regenera un
  modelo, el `id` no cambia pero sus casillas sí, y las
  [versiones](../guides/document-versions.md) no están disponibles para
  este recurso; la [actividad](./activity.md) del documento registra
  cada modificación y cada generación de fichero.

## Errores comunes

- `400 ValidationError` — `form`, `kind`, `fiscalPeriod`, `status` o
  `sortBy` llevan un valor desconocido, `fiscalYear` no es un año,
  `minDate`/`maxDate` no son una fecha `YYYY-MM-DD`, `related=records` en
  el listado, o `mode` del PDF no es `attachment`, `inline` ni `print`.
- `403 Forbidden` (`code: "token_scope_missing"`) — falta el scope
  `taxDeclarations:read`.
- `403 Forbidden` (`code: "user_role_insufficient"`) — el rol del usuario
  no tiene el permiso de impuestos.
- `404 Not Found` — el modelo no existe o está eliminado, o se pide el
  fichero de presentación de un 390.

Ver [Errores y validaciones](../guides/errors.md) para el formato
general.

## Referencia exhaustiva

Esta página cubre los matices funcionales y los casos típicos. Para el
contrato completo, consulta el
[Swagger UI](https://www.facturadirecta.com/api) o el
[openapi crudo](https://app.facturadirecta.com/openapi.json).

## Endpoints

| Método | Path | operationId | Scopes | Descripción |
|---|---|---|---|---|
| GET | `/{companyId}/taxDeclarations` | `getTaxDeclarations` | `taxDeclarations:read` | Lista de modelos de impuestos |
| GET | `/{companyId}/taxDeclarations/{id}` | `getTaxDeclaration` | `taxDeclarations:read` | Obtener un modelo de impuestos |
| GET | `/{companyId}/taxDeclarations/{id}/attachments` | `getTaxDeclarationAttachments` | `taxDeclarations:read` | Listar adjuntos de un modelo de impuestos |
| PUT | `/{companyId}/taxDeclarations/{id}/aeatFile` | `createTaxDeclarationAeatFile` | `taxDeclarations:read` | Generar el fichero de presentación de un modelo de impuestos |
| PUT | `/{companyId}/taxDeclarations/{id}/pdf` | `createTaxDeclarationPDF` | `taxDeclarations:read` | Generar el PDF de un modelo de impuestos |

## Scopes

- **`taxDeclarations:read`** — Lectura de los modelos de impuestos generados (declaraciones), sus casillas, PDF y fichero de presentación.
