---
title: Facturas rectificativas
audience:
  - developers
status: draft
---

# Facturas rectificativas

Una **factura rectificativa** corrige una factura emitida
previamente. La regula el **Art. 80 de la Ley 37/1992 del IVA**
(modificación de la base imponible) y el **RD 1496/2003** (errores
formales y materiales). FacturaDirecta da soporte completo a este
tipo de factura, incluyendo su codificación en TicketBAI, VeriFactu
y Facturae.

Esta guía cubre:

- Cuándo una factura se considera rectificativa.
- Cómo crear una rectificativa desde la API.
- Los **códigos `R1`-`R5`** y a qué supuesto del IVA corresponde
  cada uno.
- Cómo se traducen esos códigos a TicketBAI, VeriFactu y Facturae.
- El caso especial de **R5** (rectificativa de simplificadas).
- Series de rectificativas en la configuración.

Para sustitutivas (`F3`, consolidación de simplificadas), ver la
[guía de Sustitutivas](./invoices-sustitutivas.md).

## Cómo se identifica una rectificativa

FacturaDirecta detecta automáticamente si una factura es
rectificativa según tres reglas, en este orden:

1. **Si `main.correctedInvoice` está informado**, la factura es
   rectificativa **independientemente de la serie** o del total.
   `correctedInvoice` contiene el ID (`inv_<uuid>`) de la factura
   original que se está rectificando.
2. **Si la empresa tiene VeriFactu activo**, la factura es
   rectificativa **solo si** está asociada a una serie de
   rectificativas (`invoiceType: complete_correction` o
   `simplified_correction` en
   [settings de series](../sections/settings.md#lista-de-series-de-numeración)).
3. **En otro caso** (sin VeriFactu), basta con:
   - `main.total < 0` (total negativo), o
   - serie de rectificativas asociada.

La regla 1 es la que debe usar tu integración para construir
rectificativas explícitas. Las reglas 2 y 3 cubren cómo el sistema
clasifica facturas existentes.

## Cómo crear una rectificativa

No hay un endpoint dedicado: se usa el `POST /invoices` estándar
con los campos correctos.

**Pasos mínimos:**

1. Pon en `main.correctedInvoice` el ID de la factura original.
2. Elige la serie de rectificativas adecuada en `main.docNumber.series`:
   - `complete_correction` para rectificar una completa.
   - `simplified_correction` para rectificar una simplificada.
   - Si la empresa no tiene serie de rectificativas configurada,
     puedes usar la serie normal: la regla 1 sigue identificándola
     como rectificativa.
3. Si la empresa tiene **TicketBAI** activo, indica
   `main.ticketbai.claveTipoFacturaRectificativa` con el código
   `R#` que aplique (ver tabla más abajo).
4. Si la empresa tiene **VeriFactu** activo, indica
   `main.verifactu.TipoFactura` con el código `R#` que aplique.
5. Para emitir Facturae, rellena los campos
   `main.facturae.Corrective_ReasonCode`,
   `main.facturae.Corrective_CorrectionMethod` y, opcionalmente,
   `main.facturae.Corrective_AdditionalReasonDescription`.

> **No uses `main.corrective`.** Ese campo existe en el schema pero
> está marcado como reservado para uso futuro y no debe enviarse
> aún. El servidor lo ignora.

## Los códigos `R1`-`R5`

Los códigos los define la **AEAT** en la especificación de VeriFactu
y las **haciendas forales** los adoptan en TicketBAI. Son los
mismos valores con la misma semántica funcional en ambos regímenes.

| Código | Cuándo se usa |
|---|---|
| `R1` | Factura rectificativa por **error fundado en derecho** o por los supuestos del **Art. 80.1 y 80.2 LIVA** (modificación de la base imponible por causas tasadas: descuentos posteriores, devoluciones, resolución firme, etc.). |
| `R2` | Factura rectificativa por **Art. 80.3 LIVA** (concurso de acreedores). |
| `R3` | Factura rectificativa por **Art. 80.4 LIVA** (créditos incobrables: requisitos de antigüedad, reclamación, etc.). |
| `R4` | Factura rectificativa **resto** de supuestos no encajados en R1, R2 ni R3. |
| `R5` | Factura rectificativa **de una factura simplificada**. Sea cual sea el supuesto, si lo que rectificas es una simplificada, este es el código. |

Los códigos se aceptan **literalmente** en tres campos distintos
según el régimen:

| Campo | Régimen | Ámbito |
|---|---|---|
| `main.ticketbai.claveTipoFacturaRectificativa` | TicketBAI (País Vasco) | A nivel de factura. |
| `main.verifactu.TipoFactura` | VeriFactu (resto del Estado) | Reemplaza al valor por defecto `F1`/`F2`. |
| `main.facturae.Corrective_ReasonCode` | Facturae XML | Más granular (ver siguiente sección). |

> Para los supuestos exactos del Art. 80 LIVA, consulta la **Ley
> 37/1992 del IVA**. FacturaDirecta no decide si una rectificación
> encaja en R1, R2 o R3: tu integración (o el contribuyente) elige
> el código.

## Facturae

El XML Facturae usa una codificación **más granular** que TicketBAI
y VeriFactu. Cuando generas un Facturae de una rectificativa
(`PUT /invoices/{id}/facturae`), el servidor lo construye a partir
de tres campos en `main.facturae`:

### `Corrective_ReasonCode`

Código del motivo de rectificación. **Obligatorio** para emitir
Facturae de una rectificativa.

| Rango | Significado |
|---|---|
| `01`-`16` | Códigos de errores formales y materiales según el **RD 1496/2003**. Cada número corresponde a un escenario concreto (errores en la denominación, en los datos del receptor, en el importe, etc.). |
| `80`-`85` | Supuestos del **Art. 80 Ley 37/92 del IVA** (modificación de la base imponible). |

> Los códigos `R1`-`R5` (TicketBAI/VeriFactu) y los códigos
> `01`-`85` (Facturae) **no son la misma codificación**. Tu
> integración debe elegir el código adecuado para cada formato. La
> documentación oficial de Facturae (Gobierno de España, sede de
> facturae) recoge la tabla completa.

### `Corrective_CorrectionMethod`

Código del criterio empleado para la rectificación.

| Código | Significado |
|---|---|
| `02` | Rectificación por **diferencias** (la nueva factura emite la diferencia respecto a la original). |
| `03` | Rectificación por **descuento por volumen** de operaciones durante un periodo. |

### `Corrective_AdditionalReasonDescription`

Texto libre (máximo 2500 caracteres) con la ampliación del motivo:
descripción y aclaraciones de la rectificativa. Opcional pero
recomendado cuando el motivo no sea obvio del código.

## Caso especial: `R5` (simplificadas)

Cuando rectificas una **factura simplificada** (con
`main.simplified = true`), el código es siempre `R5`, sea cual sea
el supuesto regulatorio.

En TicketBAI hay un matiz adicional en Bizkaia: el sistema Batuz
genera un `Codigo` interno propio para identificar la rectificación
de simplificadas. Lo gestiona el servidor automáticamente.
Tu integración solo necesita enviar
`main.ticketbai.claveTipoFacturaRectificativa = "R5"` y enlazar
`main.correctedInvoice`.

## Series de rectificativas

Las series con `invoiceType: complete_correction` o
`simplified_correction` están configuradas en la empresa y se
pueden consultar con
[`GET /settings/series/invoice`](../sections/settings.md#lista-de-series-de-numeración).

Usar una serie de rectificativas:

- Hace que la regla de detección automática clasifique la factura
  como rectificativa incluso si `main.correctedInvoice` está vacío.
- Permite mantener una numeración separada de las facturas
  normales (típicamente con prefijo `R`).
- Es **obligatorio** en empresas con VeriFactu activo cuando no
  enlazas con `correctedInvoice` (regla 2 de detección).

Para empresas con TicketBAI sin VeriFactu, la serie es opcional:
basta con `correctedInvoice` o `total < 0` para que el sistema
clasifique correctamente.

## El campo `main.corrective`

El schema de Invoice incluye un campo booleano `main.corrective`
con la descripción:

> Reservado para uso futuro. Indicará si la factura es
> rectificativa. No utilizar aún, este campo no está activo.

**No lo envíes en tu integración**. Hoy, la rectificatividad se
detecta por las tres reglas explicadas arriba; el campo
`corrective` se activará en una fase posterior con semántica
explícita. Hasta entonces, el servidor lo ignora o lo descarta.

## Anulación vs. rectificación

Son operaciones distintas:

| Operación | Cómo se hace | Efecto fiscal |
|---|---|---|
| **Anulación** | `PUT /invoices/{id}` con `main.voided = true`. | La factura original se anula. Se contabilizan asientos de reversión. En TicketBAI/VeriFactu se envía un registro de anulación. |
| **Rectificación** | `POST /invoices` con `main.correctedInvoice` apuntando a la original. | La factura original **permanece válida**; la rectificativa la modifica (corrige importes, datos formales). |

Elige según el caso: si la factura está mal porque no debió
emitirse, anula. Si necesitas modificar importes por descuentos,
errores formales o causas del Art. 80 LIVA, rectifica.

## Errores comunes

- **Enviar `main.corrective = true`**: campo reservado, el servidor
  lo ignora. La rectificatividad se infiere por las reglas de
  detección.
- **Olvidar `claveTipoFacturaRectificativa` en TicketBAI o
  `TipoFactura: R#` en VeriFactu**: si la factura es rectificativa
  pero envías los códigos por defecto (`F1`/`F2`), el envío a la
  diputación o a AEAT puede fallar en validación.
- **Confundir `R1`-`R5` con `Corrective_ReasonCode 01`-`85`**:
  cada formato (TicketBAI/VeriFactu vs. Facturae) tiene su
  codificación. No los mezcles.
- **Usar `R5` para una completa**: `R5` es exclusivamente para
  rectificar simplificadas. Para rectificar una completa, usa
  `R1`-`R4`.
- **Crear una rectificativa sin enlazar `correctedInvoice`**: con
  VeriFactu activo, sin enlace y sin serie de rectificativas, el
  sistema no la clasificará como tal y AEAT puede rechazar el
  envío. Siempre que rectifiques una factura concreta, enlaza.

## Referencias

- **Ley 37/1992 del IVA**, Art. 80 (supuestos de modificación de la
  base imponible).
- **RD 1619/2012** (Reglamento de facturación).
- **RD 1496/2003** (códigos de errores formales/materiales para
  rectificativas).
- **Especificación de VeriFactu** (AEAT): definiciones oficiales
  de `R1`-`R5`. Ver [VeriFactu](./verifactu.md) para enlaces.
- **Documentación de TicketBAI** (haciendas forales): ver
  [TicketBAI](./ticketbai.md).
- **Especificación de Facturae**: tabla completa de
  `Corrective_ReasonCode` en la sede de facturae del Gobierno de
  España.
