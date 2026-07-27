---
title: Impuestos
audience:
  - developers
status: draft
---

# Impuestos

Los impuestos en FacturaDirecta no se envían como tasas numéricas inline:
se referencian por **identificador** desde un catálogo configurado en la
empresa. Cada empresa tiene dos catálogos separados según el documento al
que se aplique el impuesto: uno para ventas (facturas, presupuestos,
albaranes, recurrentes) y otro para compras (facturas de compra y tickets).

Esta guía cubre:

- Cómo obtener el catálogo de impuestos de una empresa.
- Forma de un impuesto (`SalesTax` / `PurchasesTax`) y sus campos clave.
- Categorías funcionales y cuándo usar cada una.
- Impuestos autorepercutidos (intracomunitarios, ISP, importación de
  servicios).
- Caso especial `lineAmountIsTax` para IVA importación DUA.
- Cómo se aplican impuestos en una línea (`line.tax`).
- Cómo se consolida en el documento (`main.taxes`).
- Recomendaciones para integradores.

## Dos catálogos: ventas y compras

Los catálogos están separados porque las operaciones de venta y compra
admiten regímenes distintos. Algunos ejemplos:

- En **ventas** existen categorías como `export` (exportación fuera de la
  UE) o `eu_consumer_vat` (IVA del país de destino para ventas a
  consumidores finales de la UE, régimen OSS). En compras no.
- En **compras** existen categorías como `import_customs` (importación
  por aduana con DUA), `import_services` (importación de servicios
  extracomunitarios) o `agriculture` (compensación al régimen especial
  agrario). En ventas no.

Mezclar catálogos no funciona: un `TaxId` del catálogo de ventas **no es
válido** en una línea de factura de compra, y viceversa.

## Obtener el catálogo

Dos endpoints, ambos bajo el scope `settings:read`:

| Endpoint | Devuelve |
|---|---|
| `GET /{companyId}/settings/taxes/sales` | Catálogo de ventas: `{ items: SalesTax[] }` |
| `GET /{companyId}/settings/taxes/purchases` | Catálogo de compras: `{ items: PurchasesTax[] }` |

Ambos aceptan un filtro opcional `taxGroup` para acotar la respuesta. Sin
paginación: devuelven el conjunto completo (ver el patrón "sin paginación"
en la [guía de paginación](./pagination.md)).

###### Copy as cURL

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/settings/taxes/sales"
```

```shell
curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://app.facturadirecta.com/api/$COMPANY_ID/settings/taxes/purchases?taxGroup=IRPF"
```

## Forma de un impuesto

Los schemas `SalesTax` y `PurchasesTax` comparten la mayoría de campos.
Las diferencias se anotan abajo.

| Campo | Tipo | Significado |
|---|---|---|
| `id` | string (`TaxId`) | Identificador estable del impuesto. Es lo que se envía en `line.tax`. |
| `value` | number | Porcentaje **en base 1**. IVA 21% → `0.21`. **No** se envía como `21`. |
| `taxGroup` | enum | Familia normativa. Ver tabla abajo. |
| `category` | enum | Categoría funcional. Determina cómo se trata el impuesto contable y fiscalmente. Ver tabla abajo. |
| `current` | boolean | `true` si está vigente actualmente. Los impuestos no vigentes existen en el catálogo para soportar documentos históricos. |
| `country` | string | Código país ISO 3166-1 Alpha-2 del país de recaudación. |
| `title` | string | Nombre completo del impuesto, legible para humanos. |
| `shortTitle` | string | Nombre corto para columnas o listas. |
| `account` | string (`AccountCode`) | Cuenta contable asociada al impuesto. |
| `description` | string | Explicación breve del propósito. |
| `selectorFlags` | array | Dónde puede aplicarse: `"product"`, `"line"`. Array **vacío** indica un impuesto autorepercutido que no se selecciona manualmente. |
| `autoApplied` | boolean | Si `true`, es un impuesto autorepercutido. No se envía en `line.tax`. |
| `pairedWith` | string (`TaxId`) | ID del impuesto pareja. En un impuesto base apunta al autorepercutido que se generará; en un autorepercutido apunta a su base. |
| `lineAmountIsTax` | boolean (solo compras) | Caso especial: el importe de la línea **es la cuota**, no la base. Aplica al IVA importación DUA. |

### Grupos (`taxGroup`)

Familias normativas. No se debe asumir que los valores son estables ante
cambios fiscales:

| Grupo | Sales | Purchases | Significado |
|---|---|---|---|
| `IVA` | ✓ | ✓ | IVA español estándar. |
| `IVA_RE` | ✓ | ✓ | IVA con recargo de equivalencia (régimen de comerciantes minoristas). |
| `IGIC` | ✓ | ✓ | Impuesto General Indirecto Canario. |
| `IPSI` | ✓ | ✓ | Impuesto sobre la Producción, Servicios e Importación (Ceuta y Melilla). |
| `IRPF` | ✓ | ✓ | Retención de IRPF. |
| `VATEU` | ✓ | — | IVA de otros países de la UE (régimen OSS). Solo aplica en ventas. |

En `VATEU`, algunos países de la UE tienen tipos legítimos del `0%` para
operaciones concretas. Si el catálogo devuelve un impuesto con `value: 0`,
trátalo como un impuesto aplicable, no como ausencia de impuesto.

En el grupo `IRPF`, un impuesto con `value: 0` —como `S_IRPF_0` o
`P_IRPF_0`, cuyo título es **Sin IRPF**— no se considera una retención
efectiva. Puedes guardar el documento sin contacto. Cualquier retención de IRPF
con un valor distinto de cero requiere un contacto asignado y válido para la
operación.

### Categorías funcionales (`category`)

La categoría es lo que controla el tratamiento real del impuesto. Misma
tabla, con qué categoría existe en cada catálogo:

| Categoría | Sales | Purchases | Significado |
|---|---|---|---|
| `standard_vat` | ✓ | ✓ | IVA / IGIC / IPSI estándar (tipos normal, reducido, superreducido y temporales). |
| `exempt_vat` | ✓ | ✓ | Operación exenta de IVA. |
| `intra_community` | ✓ | ✓ | Operación intracomunitaria. En compras genera automáticamente un par autorepercutido (efecto neto cero). |
| `reverse_charge` | ✓ | ✓ | Inversión del sujeto pasivo. El comprador liquida en lugar del vendedor. |
| `equivalence_surcharge` | ✓ | ✓ | Recargo de equivalencia. |
| `withholding` | ✓ | ✓ | Retención de IRPF general. |
| `rental_withholding` | ✓ | ✓ | Retención de IRPF por alquiler de inmueble urbano. |
| `pass_through` | ✓ | ✓ | Suplidos: gastos pagados en nombre de un tercero, sin IVA. |
| `export` | ✓ | — | Exportación a país fuera de la UE. |
| `eu_consumer_vat` | ✓ | — | IVA del país de destino para ventas B2C en la UE (régimen OSS). |
| `import_customs` | — | ✓ | Importación de bienes tramitada por aduana (DUA). El IVA se paga en aduana y es deducible. |
| `import_services` | — | ✓ | Importación de servicios extracomunitarios. Genera par autorepercutido. |
| `import_pending_customs` | — | ✓ | Importación pendiente de despacho aduanero (IVA al 0% temporal). |
| `agriculture` | — | ✓ | Compensación a explotaciones en régimen especial agrario. |

Como el conjunto de valores puede ampliarse ante cambios normativos, **no
asumas un closed set**: trata `category` como un enum extensible y degrada
gracefully con valores desconocidos.

### Impuestos autorepercutidos

Hay tres categorías que **generan automáticamente** un par de impuestos
(uno deducible y otro devengado, con efecto neto cero pero ambos
declarables):

- `intra_community` en compras (adquisición intracomunitaria de bienes).
- `reverse_charge` en compras (inversión del sujeto pasivo).
- `import_services` en compras (importación de servicios extracomunitarios).

El impuesto autorepercutido tiene `autoApplied: true` y un `selectorFlags`
vacío. **No se envía en `line.tax`.** En su lugar, envías el impuesto base
y el sistema añade automáticamente su pareja en el agregado del documento.

Ejemplo real del catálogo de compras (IVA 21% bienes corrientes
intracomunitario):

- `P_IVA_21_BC_IC` — impuesto **base** (`category: "intra_community"`,
  `autoApplied: false`, `pairedWith: "P_IVA_21_BC_IC_AR"`).
- `P_IVA_21_BC_IC_AR` — impuesto **autorepercutido** (`autoApplied: true`,
  `pairedWith: "P_IVA_21_BC_IC"`, `selectorFlags: []`).

Sufijos del ID:

- `_BC` (también `_BI`, `_SV`): bienes corrientes / bienes de inversión /
  servicios. Distingue régimen de IVA aplicable.
- `_IC`: intracomunitario. `_ISP`: inversión del sujeto pasivo.
- `_AR`: autorepercutido (siempre pareja del impuesto sin `_AR`).

Para aplicarlo en una línea:

- Envías `"tax": ["P_IVA_21_BC_IC"]`.
- En `main.taxes` del documento aparecen **ambos** impuestos consolidados,
  con `base` y `amount` simétricos (efecto neto cero).

### `lineAmountIsTax` (solo compras)

Cuando un impuesto tiene `lineAmountIsTax: true`, el comportamiento de la
línea que lo usa es especial: **el importe de la línea es directamente la
cuota de IVA**, no la base imponible. No se recalcula `base × tipo`.

Es el patrón del IVA importación tramitado por aduana (DUA), donde la
aduana repercute al importador una cuota global ya calculada. El
integrador debe detectar este flag en el catálogo y modelar la línea como
"importe = cuota", típicamente con `unitPrice` igual a la cuota notificada
en el DUA.

## Aplicar impuestos a una línea

El campo `line.tax` es **un array de `TaxId`** (strings):

```json
"lines": [
  {
    "text": "Servicios profesionales junio 2026",
    "quantity": 1,
    "unitPrice": 1500.00,
    "tax": ["S_IVA_21", "S_IRPF_15"]
  }
]
```

Notas:

- Los IDs son los `id` de los items devueltos por el endpoint del catálogo
  correspondiente a la familia del documento (sales o purchases).
- Pueden aplicarse varios impuestos a una misma línea, típicamente IVA +
  retención de IRPF para servicios profesionales.
- **No envíes impuestos con `autoApplied: true`**: el sistema los añadirá
  por su cuenta a partir del impuesto base.
- **No mezcles tipos**: si el documento es una `bill` (compra), todos los
  IDs deben venir del catálogo de compras.

El campo `taxIncludedPrices` del documento controla si `unitPrice` ya
incluye IVA. Por defecto es `false` (el `unitPrice` es base imponible).
Cuando es `true`, los importes de impuestos se calculan hacia atrás a
partir del precio total.

## Agregado del documento (`main.taxes`)

El documento incluye en `main.taxes` la consolidación por impuesto
aplicado, no la lista de impuestos por línea. Forma:

```json
"taxes": [
  {
    "id": "S_IVA_21",
    "base": 1500.00,
    "amount": 315.00,
    "companyCurrencyBase": 1500.00,
    "companyCurrencyAmount": 315.00
  },
  {
    "id": "S_IRPF_15",
    "base": 1500.00,
    "amount": -225.00,
    "companyCurrencyBase": 1500.00,
    "companyCurrencyAmount": -225.00
  }
]
```

- **`base`** es la base imponible sobre la que se calcula el impuesto.
- **`amount`** es la cuota (positiva para impuestos a pagar, negativa para
  retenciones).
- **`companyCurrencyBase`** y **`companyCurrencyAmount`** son los equivalentes
  en la moneda de la contabilidad de la empresa, útil cuando el documento
  está en divisa extranjera.
- En **purchases**, cada entrada puede llevar `nonDeductible: true` si el
  impuesto no es deducible en IRPF (campo del documento
  `nonDeductibleInIRPF`).

**Cómo se calcula:**

- Si `main.mode === "calcTotal"` (default), el agregado se **calcula
  automáticamente** a partir de las líneas. Conviene dejarlo vacío al
  crear y leerlo de la respuesta.
- Si `main.mode === "totalProvided"`, el agregado se **toma como dado**
  desde el cuerpo. Útil para importar facturas escaneadas con totales
  fijos del proveedor.

## Recomendaciones para integradores

- **Cachea el catálogo por empresa.** Los catálogos cambian con baja
  frecuencia (cambios normativos o cuando el usuario configura nuevos
  impuestos). Cachear evita una llamada extra por cada documento creado.
- **No asumas IDs estables entre empresas distintas.** El `id` es estable
  dentro de una empresa, pero dos empresas pueden tener IDs distintos
  para el "mismo" IVA 21%.
- **No asumas IDs estables en el tiempo absoluto.** Pueden definirse nuevos
  IDs ante cambios normativos. Refresca el catálogo si una llamada falla
  con `400` por TaxId desconocido.
- **Filtra por `current: true`** para ofrecer al usuario solo impuestos
  vigentes al elegir uno nuevo. Para documentos históricos, conserva los
  no vigentes.
- **Usa `selectorFlags`** para saber dónde es legítimo aplicar un
  impuesto. Un impuesto con `selectorFlags: []` solo aparece como
  autorepercutido (no se selecciona).
- **Para autorepercutidos, envía el impuesto BASE.** El sistema añadirá
  el repercutido automáticamente. Si lo intentas a la inversa, la API
  rechaza la operación.
- **Si el documento está en una divisa distinta a la de la contabilidad**,
  presta atención a los campos `companyCurrencyBase` y
  `companyCurrencyAmount` del agregado: dependen del tipo de cambio del
  documento (`exchangeRate`).

## Errores comunes

- **`400 ValidationError`** — `TaxId` no existe en el catálogo de la
  empresa. Verifica que estás consultando el catálogo correcto (sales vs
  purchases) y que el ID no es de otra empresa.
- **`400 ValidationError`** — impuesto con `autoApplied: true` enviado
  manualmente en `line.tax`. Envía el impuesto base correspondiente.
- **`400 ValidationError`** — mezcla de IDs de catálogos distintos (ID de
  ventas en un `bill`, o viceversa).
- **`400 ValidationError`** — totales enviados manualmente que no cuadran
  con las líneas en modo `calcTotal`. Cambia a `mode: "totalProvided"` o
  deja los totales vacíos.

Ver [Errores y validaciones](./errors.md) para el formato general.
