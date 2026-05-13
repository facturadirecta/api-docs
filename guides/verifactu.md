---
title: VeriFactu
audience:
  - developers
status: draft
---

# VeriFactu

**VeriFactu** es el sistema de **facturación electrónica de la
AEAT** (Agencia Estatal de Administración Tributaria) para empresas
y autónomos del Estado español que **no** estén en territorio
foral. Lo establece el Real Decreto 1007/2023, regulando los
sistemas informáticos de facturación.

Esta guía cubre cómo se integra VeriFactu con la API pública de
FacturaDirecta:

- Cuándo se activa.
- Los **dos modos** del sistema (`mode_verifactu` y
  `mode_no_verifactu`) y por qué es importante saber en cuál está
  cada factura.
- Qué campos puedes (y debes) enviar en `main.verifactu` y por
  línea.
- Qué metadatos lee tu integración en `meta.verifactu`.
- Códigos oficiales: `TipoFactura`, `CalificacionOperacion`,
  `OperacionExenta`.
- Códigos QR y URLs de verificación.
- Envío a AEAT, rate limit y reintentos.
- Anulación.
- Webhooks específicos de VeriFactu.

Para el contrato general del recurso de facturas, ver
[Facturas](../sections/invoices.md). Para el régimen equivalente del País
Vasco, ver [TicketBAI](./ticketbai.md).

## Cuándo se activa

VeriFactu se aplica cuando la **empresa emisora** está dada de alta
en VeriFactu desde su configuración fiscal. La activación es
administrativa; **no hay un flag por factura para activarlo o
desactivarlo**.

La única forma de emitir una factura sin generar VeriFactu en una
empresa con VeriFactu activo es marcarla como externa
(`main.external = true`): es el modo para registrar facturas
históricas o emitidas con otro software, **no** una vía para
saltarse la obligación.

## Los dos modos: `mode_verifactu` vs `mode_no_verifactu`

VeriFactu admite dos modos operativos. La diferencia es **cuándo y
cómo se comunican los registros a AEAT**:

| Modo | Comportamiento |
|---|---|
| **`mode_verifactu`** | Envío en **tiempo real** a AEAT. Cada factura se transmite por SOAP a la sede de Hacienda inmediatamente tras firmarla. La cadena de huellas se mantiene también. |
| **`mode_no_verifactu`** | Registros **firmados y encadenados localmente**. AEAT no recibe el envío inmediato; los registros quedan disponibles para consulta o auditoría a petición. La empresa sigue obligada a mantener el sistema de facturación íntegro. |

Tu integración no decide el modo: lo determina la configuración
fiscal de la empresa. El modo en el que se procesó **cada factura
concreta** queda registrado en `meta.verifactu.mode` (y en
`meta.verifactu.modeAnulacion` si fue anulada). Lee esos campos
para saber qué URL de verificación usar en el QR (ver más abajo).

## Campos de entrada: `main.verifactu`

Son los únicos campos de VeriFactu que tu integración puede
**enviar** en `POST /invoices` o `PUT /invoices/{id}`. El resto lo
calcula el servidor.

### Nivel factura

| Campo | Tipo | Significado |
|---|---|---|
| `TipoFactura` | enum (ver tabla) | Tipo de la factura según RD 1619/2012 y régimen rectificativo. Por defecto, el servidor lo deduce: `F1` para completas, `F2` para simplificadas. |
| `DescripcionOperacion` | string (≤ 500 chars) | Descripción libre de la operación. Útil cuando las líneas no la describen suficientemente. |
| `FacturaSimplificadaArt7273` | `"S"` o `"N"` | `"S"` si es una factura simplificada cualificada (con identificación del destinatario, Art. 7.2/7.3 del RD 1619/2012). |
| `defaultOperacionExenta` | enum `E1`-`E6` | Valor por defecto de `OperacionExenta` cuando una línea exenta no lo indica explícitamente. |

### Por línea

Cada elemento de `lines` puede llevar un sub-objeto `verifactu`:

| Campo | Tipo | Significado |
|---|---|---|
| `ClaveRegimen` | string | Clave del régimen aplicable según AEAT (`01`-`19`, dependiendo del tipo de operación). |
| `CalificacionOperacion` | enum (ver tabla) | Sólo para operaciones **sujetas** o no sujetas. Mutuamente excluyente con `OperacionExenta`. |
| `OperacionExenta` | enum `E1`-`E6` | Sólo para operaciones **exentas**. Mutuamente excluyente con `CalificacionOperacion`. |

## Códigos oficiales

Los enums los define la AEAT en la especificación técnica de
VeriFactu y los reproduce literalmente nuestro schema.

### `TipoFactura`

| Código | Significado oficial |
|---|---|
| `F1` | Factura (Art. 6, 7.2 y 7.3 del RD 1619/2012). |
| `F2` | Factura simplificada y facturas sin identificación del destinatario Art. 6.1.d) RD 1619/2012. |
| `F3` | Factura emitida en sustitución de facturas simplificadas facturadas y declaradas. |
| `R1` | Factura rectificativa (Art. 80.1 y 80.2 y error fundado en derecho). |
| `R2` | Factura rectificativa (Art. 80.3). |
| `R3` | Factura rectificativa (Art. 80.4). |
| `R4` | Factura rectificativa (resto). |
| `R5` | Factura rectificativa en facturas simplificadas. |

Detalles de cuándo usar cada `R#` en la
[guía de Rectificativas](./invoices-rectificativas.md). Para crear
una `F3` se utiliza el [endpoint dedicado de sustitutivas](../sections/invoices.md#crear-factura-sustitutiva).

### `CalificacionOperacion`

Aplica solo en líneas **sujetas** o **no sujetas** (no exentas).

| Código | Significado oficial |
|---|---|
| `S1` | Operación sujeta y no exenta — sin inversión del sujeto pasivo. |
| `S2` | Operación sujeta y no exenta — con inversión del sujeto pasivo. |
| `N1` | Operación no sujeta artículo 7, 14, otros. |
| `N2` | Operación no sujeta por reglas de localización. |

### `OperacionExenta`

Aplica solo en líneas **exentas**.

| Código | Significado oficial |
|---|---|
| `E1` | Exenta por Art. 20. |
| `E2` | Exenta por Art. 21. |
| `E3` | Exenta por Art. 22. |
| `E4` | Exenta por Art. 24. |
| `E5` | Exenta por Art. 25. |
| `E6` | Exenta otros. |

> Los códigos `E1`-`E6` de VeriFactu corresponden a artículos
> distintos de los `E1`-`E6` de TicketBAI. **No los mezcles** si
> tu integración opera con empresas en ambos regímenes.

## Metadatos: `meta.verifactu`

Tras guardar una factura definitiva (`main.draft = false`) en una
empresa con VeriFactu activo, el servidor rellena `meta.verifactu`
con la información operativa. Es **read-only**.

| Campo | Significado |
|---|---|
| `registroAlta` | Objeto con el registro de alta enviado (formato AEAT). |
| `registroAltaXml` | XML literal del registro de alta. |
| `registroAnulacion` | Objeto con el registro de anulación (cuando aplica). |
| `registroAnulacionXml` | XML literal del registro de anulación. |
| `mode` | `mode_verifactu` o `mode_no_verifactu`. Determina cómo se envió/registró el alta. |
| `modeAnulacion` | Igual que `mode` pero para el registro de anulación. |
| `qrUrl` | URL completa del código QR que debe aparecer en la representación impresa de la factura. |
| `qr` | Datos codificados del QR (útil para regenerar imagen con tu propia librería QR). |

> `registroAlta` y `registroAnulacion` son **objetos abiertos**
> (`additionalProperties: true`): siguen el contrato de AEAT, que
> puede evolucionar. Si necesitas un campo específico, accede por
> nombre; no asumas un conjunto cerrado.

## Códigos QR

Cada factura emitida con VeriFactu **debe llevar un código QR
impreso** que permite a un comprador (o a AEAT) verificar la
factura. El servidor genera la URL y los datos en
`meta.verifactu.qrUrl` y `meta.verifactu.qr`.

La URL apunta a la sede de AEAT. **Cuál de las dos URLs depende del
modo**:

| Modo de la factura | URL del QR |
|---|---|
| `mode_verifactu` | `https://www2.agenciatributaria.gob.es/wlpl/TIKE-CONT/ValidarQR?...` |
| `mode_no_verifactu` | `https://www2.agenciatributaria.gob.es/wlpl/TIKE-CONT/ValidarQRNoVerifactu?...` |

Los entornos de **preproducción** usan `prewww2.aeat.es` con el
mismo patrón de path. Si tu integración renderiza PDFs propios y no
usa nuestro generador, **respeta la URL tal cual** te llega en
`qrUrl`: el QR contiene parámetros firmados/hashados que no puedes
reconstruir.

## Envío a AEAT

Para facturas en `mode_verifactu`, el envío al webservice SOAP de
AEAT es inmediato tras guardar la factura definitiva:

1. Se construye el registro de alta a partir de la factura.
2. Se calcula la huella (hash de los campos relevantes,
   encadenando con la huella de la factura anterior).
3. Se firma el XML con el certificado de la empresa.
4. Se envía al endpoint SOAP correspondiente (producción o
   preproducción según configuración).
5. Si AEAT responde OK, el registro se considera aceptado y se
   dispara el webhook `invoice.verifactu_sent`.

Para facturas en `mode_no_verifactu`, los pasos 1-3 son los mismos
pero **no hay paso 5**: el registro queda guardado localmente con
su huella, listo para que AEAT lo consulte cuando lo requiera.

### Rate limit

AEAT puede indicar al sistema que espere antes del próximo envío
(parámetro `TiempoEsperaEnvio` en sus respuestas). FacturaDirecta
respeta ese rate limit por empresa: si AEAT pide esperar `N`
segundos, los siguientes envíos de esa empresa se posponen
automáticamente. Tu integración no necesita gestionarlo, pero
puede observar el efecto si nota un retraso entre creación de
facturas y `meta.verifactu.registroAlta` rellenado.

## Anulación

Cuando una factura se anula con `PUT /invoices/{id}` enviando
`main.voided = true`, el servidor:

1. Genera el registro de anulación.
2. Si está en `mode_verifactu`, lo envía a AEAT por SOAP.
3. Rellena `meta.verifactu.registroAnulacion` y
   `registroAnulacionXml`.
4. Guarda el modo del envío en `meta.verifactu.modeAnulacion`.
5. Emite el webhook [`invoice.voided`](../sections/invoices.md#webhooks).

## Webhooks

VeriFactu dispone de **dos eventos** relacionados:

| Evento | Cuándo se emite |
|---|---|
| `invoice.created` (general) | Al crear cualquier factura, incluido el caso VeriFactu. No es específico de VeriFactu. |
| **`invoice.verifactu_sent`** | Cuando AEAT confirma el alta de una factura concreta (solo en `mode_verifactu`). |
| **`verifactu_batch.sent`** | Cuando AEAT confirma el envío de un **lote de eventos** del sistema (eventos de facturación, sumarios periódicos). No corresponde a una factura individual. |

A diferencia de TicketBAI (que no emite webhooks específicos),
VeriFactu sí permite a tu integración reaccionar a la confirmación
de AEAT sin polling. Detalles del formato de entrega (firma HMAC,
reintentos) en [Webhooks](../sections/webhooks.md).

## Coexistencia con TicketBAI

Una factura emitida con VeriFactu **no se envía también a
TicketBAI**: los dos regímenes son excluyentes y se determinan por
la configuración fiscal de la empresa. En la práctica:

- Empresa con domicilio fiscal en País Vasco → TicketBAI (ver
  [TicketBAI](./ticketbai.md)).
- Empresa con domicilio fiscal en el resto del Estado → VeriFactu.

Si tu integración opera con empresas en ambos territorios, no
asumas la misma información: en unas facturas tendrás
`meta.verifactu` rellenado y `meta.ticketbai` ausente, y viceversa.

## Errores comunes

- **`CalificacionOperacion` y `OperacionExenta` enviados a la
  vez** en una línea: son mutuamente excluyentes. Si la operación
  es exenta, usa `OperacionExenta`. Si no, `CalificacionOperacion`.
- **Mezclar códigos de TicketBAI y VeriFactu**: los `E1`-`E6`
  refieren a artículos diferentes en cada régimen. Tu integración
  debe seleccionar el conjunto correcto según el régimen activo en
  la empresa.
- **Asumir `mode_verifactu` siempre**: si tu integración monta el
  QR en su propio PDF y elige hardcodear la URL `ValidarQR`,
  fallará para facturas en `mode_no_verifactu`. Lee siempre
  `meta.verifactu.qrUrl`.
- **No esperar al `invoice.verifactu_sent` antes de considerar
  enviada**: la respuesta del POST/PUT de la factura te confirma
  que el alta se ha registrado **localmente y se ha despachado**.
  La confirmación firmada de AEAT llega después en
  `invoice.verifactu_sent`. Si tu lógica depende de "AEAT lo tiene",
  espera al webhook.
- **Cambiar manualmente `meta.verifactu`**: es read-only; el
  servidor lo regenera al guardar. Cualquier cambio se descarta.

## Referencias oficiales

La AEAT publica la documentación técnica de VeriFactu, los esquemas
XSD, los códigos de error y la especificación del QR en su sede:

- **Portal VeriFactu**:
  [sede.agenciatributaria.gob.es](https://sede.agenciatributaria.gob.es)
  (sección "Sistemas Informáticos de Facturación y VERI*FACTU").
- **Endpoint de verificación QR** (producción):
  `https://www2.agenciatributaria.gob.es/wlpl/TIKE-CONT/ValidarQR` y
  `…/ValidarQRNoVerifactu`.
- **Endpoint de preproducción**: `https://prewww2.aeat.es/...`.

Para el significado funcional de cada código (artículos del IVA,
casos rectificativos, claves de régimen) consulta siempre la
documentación oficial: FacturaDirecta los acepta y los reenvía a
AEAT, pero la **autoridad sobre su semántica** es la Agencia
Tributaria, no este sistema.
