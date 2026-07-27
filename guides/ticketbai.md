---
title: TicketBAI
audience:
  - developers
status: draft
---

# TicketBAI

**TicketBAI** es el régimen de **facturación electrónica obligatoria
del País Vasco**. Lo regulan las tres haciendas forales —Araba,
Bizkaia y Gipuzkoa— y se aplica a toda factura de venta emitida por
empresas y autónomos con obligaciones fiscales en alguno de los tres
territorios.

Esta guía cubre cómo se integra TicketBAI con la API pública de
FacturaDirecta:

- Cuándo se activa.
- Qué campos puedes (y debes) enviar en `main.ticketbai`.
- Qué metadatos lee tu integración en `meta.ticketbai`.
- El concepto de **encadenamiento** entre facturas.
- El subsistema **Batuz** (exclusivo de Bizkaia) y el servicio
  **Zuzendu** para subsanaciones.
- Anulación y rectificativas.
- Qué información de estado no expone la API pública.

Para el contrato general del recurso de facturas, ver
[Facturas](../sections/invoices.md). Para una visión rápida del régimen
equivalente en el resto de España, ver [VeriFactu](./verifactu.md).

## Cuándo se activa

TicketBAI se aplica cuando la **empresa emisora** está dada de alta
en TicketBAI en su diputación foral. La configuración la determina
el alta administrativa de la empresa; **no hay un flag por factura
para activarlo o desactivarlo**.

La única forma de emitir una factura **sin generar TicketBAI** en
una empresa con TicketBAI activo es marcarla como externa
(`main.external = true`). Esa marca indica que la factura se generó
fuera del programa y contabiliza normalmente, pero **no produce
información para TicketBAI ni VeriFactu**. Es el modo apropiado para
registrar facturas históricas o emitidas con otro software, no para
saltarse la obligación electrónica.

### Diputación foral

La factura registra su territorio en `meta.ticketbai.diputacionForal`,
con uno de los tres valores:

| Valor | Diputación |
|---|---|
| `Araba` | Diputación Foral de Álava |
| `Bizkaia` | Diputación Foral de Bizkaia |
| `Gipuzkoa` | Diputación Foral de Gipuzkoa |

Lo decide el sistema según la configuración fiscal de la empresa;
tu integración solo lo lee.

## Bizkaia es distinto: Batuz

Bizkaia opera un sistema adicional llamado **Batuz**: el LROE (Libro
Registro de Operaciones Económicas) que recibe los XML de ingresos
y gastos. Por eso, una factura emitida en Bizkaia genera **dos**
documentos XML: el de TicketBAI estándar y el de Batuz.

Implicaciones para tu integración:

- En Bizkaia tendrás campos `batuz*` adicionales en
  `meta.ticketbai` (`batuzAccepted`, `batuzAcceptedWithErrors`,
  `batuzAcceptedZuzendu`, `batuzModel`, `batuzModelZuzendu`).
- Existe un campo de entrada **`batuzRentaIngresos`** en
  `main.ticketbai` para indicar el epígrafe de IRPF de la operación
  (obligatorio en algunos escenarios de Bizkaia).
- Si la factura se emitió originalmente **sin Software Garante**
  (papel, otro software no certificado) y se sube a Bizkaia para
  rectificarla, la información llega en un meta alternativo
  (`InvoiceMetaBatuzNoSG`) en vez del estándar (`InvoiceMetaTicketbai`).
  Tu integración debe distinguir cuál de los dos viene.

En Araba y Gipuzkoa, los campos `batuz*` no aplican.

## Campos de entrada: `main.ticketbai`

Son los únicos campos de TicketBAI que tu integración puede
**enviar** en `POST /invoices` o `PUT /invoices/{id}`. El resto lo
calcula el servidor.

| Campo | Tipo | Cuándo se usa |
|---|---|---|
| `causaExencion` | enum `E1`-`E6` | Cuando la operación está exenta de IVA, indicas el motivo según la clasificación oficial de las diputaciones forales. |
| `causaNoSujeta` | enum `OT`, `RL`, `VT`, `IE` | Cuando la operación no está sujeta a IVA, indicas el motivo. |
| `claveTipoFacturaRectificativa` | enum `R1`-`R5` | Solo en facturas rectificativas. Identifica el tipo de rectificativa. Ver [Rectificativas](./invoices-rectificativas.md) para qué corresponde a cada `R#`. |
| `batuzRentaIngresos` | array de 1 elemento `{ epigrafe: string }` | Solo Bizkaia. `epigrafe` es el código de epígrafe IRPF (1-7 caracteres). |

> Los códigos `E1`-`E6`, `OT`/`RL`/`VT`/`IE` y `R1`-`R5` son los
> oficiales de las haciendas forales. Su significado funcional está
> en la documentación técnica enlazada al final de esta página, no
> en el código de FacturaDirecta. Antes de enviar uno, **verifica
> en la fuente oficial** que tu operación encaja en esa categoría.

Si tu integración no envía nada de esto, el servidor genera el XML
con los valores por defecto que se hayan calculado a partir de los
datos de la factura (régimen, líneas, contacto). Solo se rellenan
estos campos cuando tienes una operación atípica que requiere
indicarlo explícitamente.

## Metadatos: `meta.ticketbai`

Tras guardar una factura definitiva (`main.draft = false`) en una
empresa con TicketBAI activo, el servidor rellena `meta.ticketbai`
con la información operativa. Es **read-only**: tu integración
**lee**, no escribe.

### Campos principales

| Campo | Significado |
|---|---|
| `identificador` | Identificador TicketBAI de la factura. Se genera siguiendo el formato de la diputación correspondiente. |
| `ordenEncadenamiento` | Entero ≥ 0. Posición de la factura en la cadena de TicketBAI de la empresa (ver [Encadenamiento](#encadenamiento)). |
| `diputacionForal` | `Araba` / `Bizkaia` / `Gipuzkoa` (ver tabla más arriba). |
| `mustBeSent` | Campo obsoleto conservado por compatibilidad. Siempre vale `false`; no indica si existe un envío pendiente. |
| `sent` | Campo de compatibilidad. En los documentos procesados por el flujo actual permanece en `false`; no indica el resultado del envío. |
| `qrUrl`, `qr` | URL y datos del código QR que debe aparecer en la representación impresa. |
| `xml` | XML de **alta** firmado y enviado a TicketBAI. |
| `encadenamientoFactura` | Referencia a la factura inmediatamente anterior en la cadena (ver siguiente sección). |

### Campos exclusivos de Bizkaia (Batuz)

| Campo | Significado |
|---|---|
| `batuzAccepted` | Campo de compatibilidad que no refleja el estado de los envíos procesados por el flujo actual. |
| `batuzAcceptedWithErrors` | Campo de compatibilidad que no refleja si el envío actual se aceptó con avisos. |
| `batuzAcceptedZuzendu` | Campo de compatibilidad que no refleja el resultado actual de Zuzendu. Ver [Zuzendu](#zuzendu-bizkaia). |
| `batuzModel` | Modelo interno del XML Batuz de alta. No procesar como contrato estable. |
| `xmlNoSG` | XML de Batuz cuando la factura se emitió originalmente sin Software Garante. |

### Campos de anulación y rectificación

| Campo | Cuándo aparece |
|---|---|
| `voided` | La factura ha sido anulada. Contiene el XML de anulación y campos de compatibilidad `mustBeSent`/`sent`, ambos inertes para consultar el estado. |
| `zuzenduAlta` | La factura ha sido **subsanada** vía Zuzendu (correcciones a una factura previa sin emitir una rectificativa). |

> Los campos `xmlModel`, `batuzModel`, `batuzModelZuzendu` contienen
> el **modelo interno** desde el que el servidor genera el XML. Sus
> claves no son parte del contrato público y pueden cambiar entre
> versiones. Si necesitas inspeccionar el XML, usa el campo `xml`
> (string) o `xmlNoSG`.

## Encadenamiento

TicketBAI exige que **cada factura referencie a la inmediatamente
anterior** emitida por la misma empresa, creando una cadena
verificable de extremo a extremo. Es la pieza que da resistencia al
sistema: no se puede modificar una factura intermedia sin romper la
cadena posterior.

El servidor calcula automáticamente la referencia y la guarda en
`meta.ticketbai.encadenamientoFactura`:

| Campo | Significado |
|---|---|
| `SerieFacturaAnterior` | Serie de la factura previa (puede no estar si la previa no la tenía). |
| `NumFacturaAnterior` | Número de la factura previa. |
| `FechaExpedicionFacturaAnterior` | Fecha de emisión de la factura previa. |
| `SignatureValueFirmaFacturaAnterior` | Valor de la firma electrónica de la factura previa. |

Tu integración no escribe estos campos. Pero si los lees (para una
herramienta de auditoría o verificación), conviene saber:

- El orden lo determina la **expedición**, no la creación. El campo
  `meta.expeditionTimestamp` de la factura marca el instante en que
  pasó de provisional a definitiva.
- `meta.ticketbai.ordenEncadenamiento` te da el índice numérico
  dentro de la cadena de la empresa.

## Envío y reintentos

El envío a TicketBAI (y a Batuz en Bizkaia) se inicia **al guardar
la factura como definitiva**:

1. Se genera el XML.
2. Se firma con el certificado de la empresa.
3. Se envía a la diputación correspondiente.
4. Si la respuesta indica un error técnico, como un timeout o una
   indisponibilidad temporal, FacturaDirecta reintenta el envío con
   **backoff exponencial**.
5. Pasadas **24 horas** desde el primer error sin éxito, los reintentos
   automáticos cesan.

Los campos `mustBeSent`, `sent` y `batuzAccepted*` de `meta.ticketbai` no
se actualizan con el resultado. Se conservan por compatibilidad y no debes
usarlos para monitorizar el envío.

> **No hay webhooks específicos de TicketBAI.** A diferencia de
> VeriFactu, que emite [`invoice.verifactu_sent`](../sections/webhooks.md),
> TicketBAI no genera eventos propios; tu integración debe leer
> `meta.ticketbai` cuando le interese conocer el estado.

### Corregir un gasto rechazado por Batuz

Si Batuz rechaza el alta de una factura de compra o un ticket y nunca lo
acepta, puedes corregir los datos identificativos mediante
`PUT /bills/{id}`. Esto incluye el proveedor, su identificación fiscal, el
número de factura y sus fechas. Al guardar, el servidor genera otro alta.

Si Batuz aceptó el documento alguna vez, esos datos quedan protegidos. Para
cambiarlos, anula el documento y crea otro. Esta protección también se aplica
si el último intento fue rechazado pero existe una aceptación anterior.

## Anulación

Cuando una factura se anula con `PUT /invoices/{id}` enviando
`main.voided = true`, el servidor:

1. Genera el XML de anulación.
2. Lo firma y lo envía a la diputación (y a Batuz en Bizkaia).
3. Rellena `meta.ticketbai.voided` con el resultado del envío.
4. Emite el webhook [`invoice.voided`](../sections/invoices.md#webhooks).

La factura **no desaparece**: queda en estado `voided`. La anulación tiene su
propia cola de reintentos. El campo `meta.ticketbai.voided.sent` no refleja el
resultado del flujo actual.

## Rectificativas

Las **facturas rectificativas** se crean con el CRUD normal de
facturas (`POST /invoices`), enlazando con `main.correctedInvoice` a
la factura original y rellenando
`main.ticketbai.claveTipoFacturaRectificativa` con el código
`R1`-`R5` apropiado.

Hay un caso especial: cuando rectificas una **factura simplificada**
desde Bizkaia con `claveTipoFacturaRectificativa = "R5"`, el
sistema usa un flujo distinto. Detalles, codificación de cada `R#`
y ejemplos en la [guía de Rectificativas](./invoices-rectificativas.md).

## Zuzendu (Bizkaia)

**Zuzendu** es el servicio de Bizkaia que permite **subsanar** una
factura ya enviada **sin emitir una rectificativa**. El típico caso:
un error tipográfico en el concepto o en una dirección, que no
afecta a importes ni a la naturaleza de la operación.

Cuando una factura se subsana vía Zuzendu, el servidor:

1. Genera un XML de alta Zuzendu (con la corrección).
2. Lo envía a Batuz.
3. Rellena `meta.ticketbai.zuzenduAlta` con el `xmlModel` y `xml`
   correspondientes.
4. FacturaDirecta conserva el resultado fiscal fuera de los campos de
   compatibilidad de `meta.ticketbai`.

Para facturas que se emitieron originalmente **fuera de
FacturaDirecta** (sin Software Garante), el meta es distinto:
`InvoiceMetaBatuzNoSG` en lugar de `InvoiceMetaTicketbai`. Tu
integración debe comprobar la presencia de `meta.batuzNoSG` o
similar para detectar este caso antes de procesar campos que solo
existen en uno u otro.

## Consultar el estado desde una integración

La API pública no expone el estado autoritativo de los envíos TicketBAI. Tampoco
hay webhooks específicos. Por tanto, no puedes confirmar una aceptación o un
rechazo mediante polling de `meta.ticketbai`.

Puedes leer los artefactos de compatibilidad, como el XML generado, pero no uses
`mustBeSent`, `sent`, `voided.sent` ni `batuzAccepted*` para tomar decisiones.
Si tu integración necesita conocer el resultado fiscal, esa capacidad no está
disponible en el contrato público actual.

## Coexistencia con VeriFactu

Una factura emitida con TicketBAI **no se envía también a VeriFactu**:
los dos regímenes son excluyentes y se determinan por la
configuración fiscal de la empresa. En la práctica:

- Empresa con domicilio fiscal en País Vasco → TicketBAI.
- Empresa con domicilio fiscal en resto del Estado → VeriFactu (ver
  [VeriFactu](./verifactu.md)).

Si tu integración opera con empresas mixtas, no asumas el mismo
flujo: en las páginas de factura tendrás `meta.ticketbai` rellenado
en unas y `meta.verifactu` en otras.

## Errores comunes

- Enviar una factura cuyo cliente tiene un país que no pertenece al catálogo
  admitido por el esquema de TicketBAI. El servidor rechaza la operación antes
  de generar el XML e indica el país y su código. Actualiza el contacto con un
  código válido. En Bizkaia, la misma validación se aplica al contacto de los
  gastos enviados al Libro de Registro de Operaciones Económicas (LROE).
- Enviar `claveTipoFacturaRectificativa` sin que la factura sea
  realmente rectificativa (no enlazada con `main.correctedInvoice` o
  no usando una serie de rectificativas). El XML se generará pero la
  diputación rechazará el envío en validación.
- Olvidar `batuzRentaIngresos` en Bizkaia en operaciones donde es
  obligatorio. La diputación devuelve error de validación en el
  envío.
- Asumir que `meta.ticketbai.sent = true` es inmediato tras `POST`.
  Si la diputación está sobrecargada, puede tardar minutos; tu
  integración no debe bloquearse esperándolo.
- Procesar el contenido de `xmlModel` (modelo interno) en lugar de
  `xml` (XML literal). El modelo interno no es contrato estable.

## Referencias oficiales

Las haciendas forales publican la documentación técnica completa
(esquemas XSD, códigos de error, validaciones) en sus webs:

- **Araba**: [web.araba.eus/es/hacienda/ticketbai/documentacion-tecnica](https://web.araba.eus/es/hacienda/ticketbai/documentacion-tecnica)
- **Bizkaia (Batuz)**: [www.batuz.eus](https://www.batuz.eus)
- **Gipuzkoa**: [www.gipuzkoa.eus/es/web/ogasuna/ticketbai/documentacion-y-normativa](https://www.gipuzkoa.eus/es/web/ogasuna/ticketbai/documentacion-y-normativa)

Para el significado funcional de los códigos
(`E1`-`E6`, `OT`/`RL`/`VT`/`IE`, `R1`-`R5`) consulta siempre la
documentación oficial: FacturaDirecta los acepta y los reenvía a la
diputación, pero la **autoridad sobre su semántica** es la
hacienda foral, no este sistema.
