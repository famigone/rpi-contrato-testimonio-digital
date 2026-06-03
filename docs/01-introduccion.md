# 01 — Introducción

## Qué es el testimonio digital

El **testimonio digital** es la versión digital nativa de los testimonios
notariales que hoy se presentan en papel al RPI de Neuquén. En lugar de que el
escribano imprima el testimonio en papel, lo firme con su firma ológrafa, lo
acompañe con una minuta y lo presente físicamente por Mesa de Entradas, el
testimonio viaja desde el sistema del Colegio al RPI **completamente por API**,
firmado digitalmente.

Un testimonio digital es:

- Un **XML estructurado** con los datos del acto (escribano, partes, inmueble,
  monto, etc.), firmado por el escribano autorizante con su certificado digital.
- Un **PDF firmado** del testimonio en sí, que es el documento legal con valor
  de testimonio.

Ambas piezas se envían juntas al RPI en una sola petición HTTP.

## Qué problema resuelve

El flujo actual en papel implica:

- Impresión del testimonio.
- Confección y impresión de la minuta.
- Presentación física en Mesa de Entradas.
- Generación de código de barras y escaneo posterior por operador del RPI.
- Carga de datos en el sistema registral.

El testimonio digital elimina la impresión, el desplazamiento físico y la carga
manual. Los datos llegan estructurados al RPI y se procesan automáticamente.

## Alcance de la v1.0

La primera versión del contrato cubre **un solo tipo de acto**:

- **Compraventa** de inmuebles, escritura notarial.

En versiones futuras se agregarán otros actos (hipoteca, donación, división de
condominio, etc.) sin cambiar la estructura general del XML — solo se agregarán
nuevos archivos XSD bajo `xsd/actos/`.

A diferencia del borrador inicial, esta versión **sí** soporta:

- **Personas jurídicas** (sociedades, asociaciones) y **organismos públicos**
  como adquirentes o transmitentes, además de personas humanas.
- **Representantes** (tutor, apoderado, etc.) mediante un bloque opcional dentro
  de cada persona.

Tampoco están soportadas en v1.0:

- **Asentimiento conyugal estructurado**: se modela como **texto libre** en el
  campo `AsentimientoConyugal`, no como bloque con cónyuge, fecha, etc.
- **Múltiples actos** en un mismo testimonio (un testimonio = un acto).
- **Otros tipos de actos** (hipoteca, donación, división de condominio, etc.):
  se incorporarán en versiones futuras.
- **Ampliatorios** o reingresos del mismo testimonio (cada envío genera un
  trámite nuevo).

## Roles en la integración

| Rol | Quién lo cumple | Responsabilidad |
|-----|-----------------|-----------------|
| Emisor | Sistema del Colegio de Escribanos | Genera el XML, lo firma con el certificado del escribano, lo envía al RPI |
| Receptor | Servicio API del RPI | Valida XML y firma, persiste, sincroniza con sistema registral |
| Firmante | Escribano autorizante | Es quien firma digitalmente el XML con su certificado personal |
| Notificador | Servicio API del RPI | Devuelve al Colegio los cambios de estado del testimonio (provisorio, definitivo, rechazo) |

**El RPI no conoce a los escribanos individualmente como interlocutores HTTP.**
La identidad del escribano viaja embebida en el XML (CUIT, nombre, registro).
El interlocutor HTTP es siempre el sistema del Colegio.

## Próximos pasos

Si querés entender cómo funciona el flujo completo, seguí con
[02 — Flujo end-to-end](02-flujo-end-to-end.md).

Si querés ir directo al contrato técnico, andá a
[03 — Endpoint API](03-endpoint-api.md).
