---
title: Facturas sustitutivas (F3)
audience:
  - developers
status: draft
---

# Facturas sustitutivas (F3)

Una **factura sustitutiva** es una factura completa que reemplaza
a un conjunto de **facturas simplificadas** previamente emitidas.
En VeriFactu corresponde al tipo **`F3`**, definido por AEAT como:

> Factura emitida en sustitución de facturas simplificadas
> facturadas y declaradas.

El caso de uso típico es un comercio que emite tickets
(simplificadas) durante el mes y, a final de mes, un cliente le
pide una factura completa con sus datos fiscales agrupando varios
tickets. La sustitutiva consolida esos tickets en un único
documento con destinatario identificado.

Esta guía cubre:

- Por qué la sustitutiva tiene un endpoint dedicado y no se crea
  con el `POST /invoices` normal.
- Las **6 validaciones** que aplica el servidor.
- Cómo se construyen automáticamente las líneas.
- Diferencia con rectificación.

Para rectificativas (`R1`-`R5`), ver
[Rectificativas](./invoices-rectificativas.md).

## Endpoint dedicado: `POST /invoices/substitution`

A diferencia del CRUD normal de facturas, las sustitutivas tienen
**un endpoint propio**:

```
POST /{companyId}/invoices/substitution
```

Razón del endpoint dedicado: la construcción del cuerpo de la
factura (líneas, totales, separadores, referencias a las
originales) la hace **el servidor**. Tu integración solo envía
**qué simplificadas se sustituyen** y **a quién se le factura**;
el resto se deriva.

Por la misma razón, el `POST /invoices` estándar **rechaza** que le
envíes `main.substitution = true`: devuelve `400` con mensaje
"Substitution invoices cannot be created through the public API".
La única vía es el endpoint dedicado.

## Body del request

| Campo | Tipo | Requerido | Significado |
|---|---|---|---|
| `contact` | string | Sí | ID del cliente (`con_*`) al que se emite la factura completa. |
| `substitutedInvoices` | string[] | Sí, no vacío | UUIDs (`inv_*`) de las facturas simplificadas a sustituir. Sin duplicados. |
| `docNumber` | `{ series, formattedSeries?, number? }` | No | Si no lo envías, el servidor elige la primera serie no oculta de tipo `complete` o `simplified` configurada en la empresa. |
| `date` | date (`YYYY-MM-DD`) | No | Fecha de la factura sustitutiva. Por defecto, hoy. |
| `notes` | string | No | Notas visibles. Se prefijan automáticamente con `"Factura emitida en sustitución de facturas simplificadas"`. |
| `internalNotes` | string | No | Notas internas (no se imprimen). |
| `tags` | string[] | No | Etiquetas. |

No envías líneas: las genera el servidor (ver siguiente sección).

## Validaciones del servidor

Antes de crear la sustitutiva, el servidor aplica las siguientes
validaciones sobre las facturas referenciadas en
`substitutedInvoices`. **Cualquier fallo devuelve `400`** con un
mensaje explicativo.

| # | Regla | Mensaje de error si falla |
|---|---|---|
| 1 | Todas existen y no están archivadas | `Invoices not found: <ids>` |
| 2 | Todas son simplificadas (`main.simplified === true`) | `Invoice <id> is not a simplified invoice` |
| 3 | Ninguna es externa (`main.external !== true`) | `Invoice <id> is an external invoice and cannot be substituted` |
| 4 | Todas comparten la misma `currency` | `All substituted invoices must have the same currency` |
| 5 | Todas comparten el mismo `taxIncludedPrices` | `All substituted invoices must have the same taxIncludedPrices value` |
| 6 | Si alguna tiene `contact`, **todas las que lo tengan** deben tener el mismo | `Substituted invoices with contacts must all have the same contact` |
| 7 | Ninguna está ya sustituida en otra F3 | `Some invoices are already substituted: <details>` |

> La regla 6 admite mezcla: simplificadas sin contacto + algunas
> con contacto se aceptan, **siempre que todas las que tengan
> contacto compartan el mismo `con_*`**. El `contact` del request
> (cliente al que se factura la sustitutiva) no tiene por qué
> coincidir con esos contactos previos.

## Cómo se construyen las líneas

El servidor genera las líneas del cuerpo de la sustitutiva
recorriendo `substitutedInvoices` **en el orden enviado**. Para
cada simplificada:

1. **Inserta una línea separadora** con
   `quantity = 0`, `unitPrice = 0`, `lineTotal = 0`, sin
   impuestos, con `text` autogenerado del formato
   `"<docNumber> (<fecha>)"` y `origin` apuntando al `inv_*` de la
   simplificada. Esa línea cero hace de etiqueta visual del grupo.
2. **Copia todas las líneas** de la simplificada,
   anotando `origin` con el `inv_*` de la original para trazar la
   procedencia.

El resultado es una factura completa con bloques de líneas, cada
bloque encabezado por una línea-separadora. El campo `origin` de
cada línea permite reconstruir qué simplificada generó qué línea.

Además, el servidor:

- Pone `main.substitution = true`.
- Rellena `main.substitutedInvoices` con los UUIDs.
- Deduce `currency` y `taxIncludedPrices` de las simplificadas.
- Genera VeriFactu con `TipoFactura: F3` automáticamente. No tienes
  que indicarlo en el request.

## Efecto contable y fiscal

A diferencia de las facturas normales, la sustitutiva **no genera
contabilidad propia**. Las simplificadas originales ya generaron
sus asientos en el momento en que se emitieron; la sustitutiva
**no duplica esos asientos**: actúa como una agrupación
documental.

Su función principal es:

- Dotar al cliente de una factura completa para su deducibilidad.
- Generar la información para el **modelo 347** (declaración
  informativa de operaciones con terceros), agregando las
  operaciones a un único cliente.

Para los flujos de cobro, **los cobros se siguen registrando en las
simplificadas originales**, no en la sustitutiva. La sustitutiva no
tiene saldo pendiente: nace consolidada.

## Sustitutiva vs. rectificativa

Son operaciones distintas que no deben confundirse:

| | Sustitutiva (`F3`) | Rectificativa (`R1`-`R5`) |
|---|---|---|
| **Propósito** | Agrupar varias simplificadas en una completa para identificar al destinatario. | Corregir una factura previa por errores formales o por Art. 80 LIVA. |
| **Documentos involucrados** | Una sustitutiva nueva + N simplificadas originales (válidas). | Una rectificativa nueva + la factura original (válida o corregida). |
| **Endpoint** | `POST /invoices/substitution` (dedicado). | `POST /invoices` con `main.correctedInvoice`. |
| **Efecto contable** | Ninguno; no duplica asientos. | Genera asientos correctivos. |
| **Codificación VeriFactu** | `TipoFactura: F3`, automático. | `TipoFactura: R1`-`R5`, lo eliges tú. |

Si lo que quieres es **corregir** una simplificada porque tenía un
error, no la sustituyas: ver [Rectificativas](./invoices-rectificativas.md).

## Ejemplo

###### Request

```json
{
  "contact": "con_2e3f4a18-9b7c-4d6a-8e1f-5c2d3b4a7e9f",
  "substitutedInvoices": [
    "inv_3a7b9c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c",
    "inv_4b8c0d15-3a6f-4b8c-c1d9-2e3f4a5b6c7d",
    "inv_5c9d1e16-4b7a-4c9d-d2e0-3f4a5b6c7d8e"
  ],
  "date": "2026-05-31",
  "notes": "Factura agrupada del mes de mayo."
}
```

###### cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" \
  -d @substitution.json \
  "https://app.facturadirecta.com/api/$COMPANY_ID/invoices/substitution"
```

###### Respuesta (resumida)

```json
{
  "content": {
    "uuid": "inv_8f2a3b4c-5d6e-4a7b-9c0d-1e2f3a4b5c6d",
    "type": "invoice",
    "main": {
      "contact": "con_2e3f4a18-9b7c-4d6a-8e1f-5c2d3b4a7e9f",
      "substitution": true,
      "substitutedInvoices": [
        "inv_3a7b9c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c",
        "inv_4b8c0d15-3a6f-4b8c-c1d9-2e3f4a5b6c7d",
        "inv_5c9d1e16-4b7a-4c9d-d2e0-3f4a5b6c7d8e"
      ],
      "verifactu": { "TipoFactura": "F3" },
      "notes": "Factura emitida en sustitución de facturas simplificadas\nFactura agrupada del mes de mayo.",
      "lines": [
        { "text": "F-26-101 (05 may 2026)", "quantity": 0, "unitPrice": 0, "lineTotal": 0, "tax": [], "origin": "inv_3a7b9c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c" },
        { "text": "Producto A", "quantity": 2, "unitPrice": 50, "lineTotal": 100, "tax": ["S_IVA_21"], "origin": "inv_3a7b9c14-2f5e-4a7b-b0c8-1d2e3f4a5b6c" },
        { "text": "F-26-115 (12 may 2026)", "quantity": 0, "unitPrice": 0, "lineTotal": 0, "tax": [], "origin": "inv_4b8c0d15-3a6f-4b8c-c1d9-2e3f4a5b6c7d" }
      ]
    }
  }
}
```

## Errores comunes

- **Intentar crear una sustitutiva con `POST /invoices`** enviando
  `main.substitution = true`: el endpoint normal lo rechaza con
  `400`. Usa el endpoint dedicado.
- **Sustituir una factura ya completa** (no simplificada): el
  servidor rechaza con la regla 2. La sustitutiva es para
  consolidar simplificadas, no completas.
- **Sustituir simplificadas externas**: la regla 3 lo rechaza. Las
  externas no se envían a VeriFactu, así que no se puede generar
  una `F3` que las consolide.
- **Mezclar monedas o `taxIncludedPrices`**: reglas 4 y 5.
  Resuélvelo emitiendo varias sustitutivas separadas, una por
  moneda/configuración.
- **Re-sustituir** una simplificada ya incluida en otra `F3`: la
  regla 7 lo rechaza. Si necesitas modificar una sustitutiva ya
  emitida, anúlala y emite una nueva.

## Referencias

- **Reglamento de facturación (RD 1619/2012)**, Art. 6.1.d) y
  Art. 7 (simplificadas) y supuestos de factura completa.
- **Especificación de VeriFactu** (AEAT): definición oficial de
  `TipoFactura: F3` y casos de uso. Ver
  [VeriFactu](./verifactu.md) para enlaces.
- Para el ciclo de vida del recurso, ver el
  [endpoint de creación](../sections/invoices.md#crear-factura-sustitutiva)
  en la página de Facturas.
