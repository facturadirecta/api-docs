---
title: API Pública de FacturaDirecta
description: >-
  Conecta tu software con FacturaDirecta mediante la API REST pública. Accede a contactos,
  productos, facturas, presupuestos, albaranes, gastos, bancos, webhooks y más.
audience:
  - developers
status: draft
---

# API Pública de FacturaDirecta

<!-- source: api.ts -->

La **API pública de FacturaDirecta** te permite integrar tu software con el
sistema de facturación. Puedes automatizar procesos como crear facturas desde un
CRM, sincronizar contactos con un ERP, conectar una tienda online o generar
informes personalizados con datos de facturación y contabilidad.

La API es REST, usa JSON y se documenta mediante OpenAPI.

> **Recomendación**: antes de empezar a desarrollar, crea un [entorno de pruebas
> (sandbox)](https://help.facturadirecta.com/es/articles/15157082-entorno-de-pruebas-sandbox)
> para hacer tus primeras llamadas sin afectar a los datos reales de tu empresa.

## URL base y referencia OpenAPI

<!-- source: api.ts -->

Todas las URLs de la API pública en producción tienen este prefijo:

```text
https://app.facturadirecta.com/api
```

Recursos técnicos:

- **Swagger UI / referencia navegable**: `https://www.facturadirecta.com/api`
- **OpenAPI crudo**: `https://app.facturadirecta.com/openapi.json`

Puedes importar el OpenAPI en herramientas como Postman o Insomnia, o generar
clientes automáticamente con herramientas tipo OpenAPI Generator.

## Autenticación

<!-- source: api.ts -->

La API admite dos métodos de autenticación:

### API key

Recomendado para integraciones de una empresa concreta: scripts internos,
conectores, plugins o automatizaciones.

Crea la clave desde FacturaDirecta y envíala en la cabecera:

```http
facturadirecta-api-key: TU_CLAVE_API
```

Ventajas:

- El acceso se limita a los permisos asignados a la clave.
- Cada clave pertenece a la empresa donde se creó.
- No depende de una sesión de usuario.

Consulta [API keys](./api-keys.md) para la referencia técnica y
[API keys en Ajustes](https://help.facturadirecta.com/es/articles/15157077-api-keys)
para crear la clave desde la interfaz.

### OAuth2

Recomendado para aplicaciones que conectan con empresas de terceros o necesitan
que un usuario autorice scopes concretos.

Usa el flujo Authorization Code con PKCE y envía el token en la cabecera:

```http
Authorization: Bearer access_token
```

Consulta [Autenticación](./authentication.md) para el detalle del flujo OAuth2,
scopes y endpoints de autorización.

## Recursos disponibles

<!-- source: api-docs-map -->

La API ofrece acceso, entre otros, a estos recursos:

| Recurso | Para qué sirve |
|---|---|
| [Perfil](../sections/profile.md) | Información del usuario autenticado y empresas accesibles. |
| [Contactos](../sections/contacts.md) | Clientes, proveedores, empleados y otros contactos. |
| [Productos](../sections/products.md) | Catálogo de productos y servicios. |
| [Facturas de venta](../sections/invoices.md) | Crear, consultar, actualizar, enviar, anular o descargar facturas. |
| [Facturas recurrentes](../sections/recurring.md) | Automatizaciones de facturación recurrente. |
| [Presupuestos](../sections/estimates.md) | Presupuestos y proformas. |
| [Albaranes](../sections/delivery-notes.md) | Albaranes de entrega. |
| [Facturas de compra y tickets](../sections/bills.md) | Gastos, compras y tickets. |
| [Nóminas](../sections/payrolls.md) | Nóminas de empleados. |
| [Bancos](../sections/banks.md) | Cuentas bancarias y tesorería. |
| [Extractos bancarios](../sections/statements.md) | Extractos y conciliación. |
| [Métodos de pago](../sections/payment-methods.md) | Métodos de cobro y pago. |
| [Diario contable](https://help.facturadirecta.com/es/articles/15084462-diario-contable) | Asientos y movimientos contables. |
| [Configuración](../sections/settings.md) | Series, impuestos, plantillas y configuración fiscal. |
| [Bandeja de entrada](../sections/inbox.md) | Procesamiento de documentos recibidos. |
| [Uploads](../sections/uploads.md) | Subida de archivos. |
| [Webhooks](https://help.facturadirecta.com/es/articles/13905261-webhooks) | Endpoints y eventos para recibir cambios en tiempo real. |

> **API pública y Zapier no tienen el mismo alcance**: Zapier expone solo
> algunas acciones de FacturaDirecta. Aunque la API pública incluye
> **Albaranes**, Zapier no permite crear albaranes. Usa la API pública
> directamente si necesitas automatizar ese recurso.

## Webhooks

<!-- source: webhookDispatcher -->

Los **webhooks** permiten recibir notificaciones en tiempo real cuando ocurre un
cambio relevante en tu empresa: una factura creada, un contacto modificado, un
gasto archivado, etc.

En lugar de consultar la API periódicamente, configuras una URL HTTPS y
FacturaDirecta envía un `POST` a tu servidor cuando se produce un evento al que
estás suscrito.

Los webhooks están disponibles en los planes **Avanzado** y **Total**.

Consulta:

- [Webhooks](https://help.facturadirecta.com/es/articles/13905261-webhooks) — payload, catálogo de eventos, firma y referencia
  técnica de endpoints/eventos.
- [Configurar webhooks](https://help.facturadirecta.com/es/articles/15097369-configurar-webhooks)
  — cómo crear endpoints desde Ajustes.

## Primeros pasos recomendados

1. Crea o selecciona un sandbox.
2. Crea una API key con los permisos mínimos necesarios.
3. Haz una primera llamada de lectura, por ejemplo listar facturas o contactos.
4. Prueba una operación de escritura en sandbox.
5. Si tu integración necesita reaccionar a cambios, configura webhooks.

Consulta [Inicio rápido con la API](./quickstart.md) para una primera conexión
paso a paso.
