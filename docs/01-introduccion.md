# 01 — Introducción

## Qué es el testimonio digital

El **testimonio digital** es la versión digital nativa de los testimonios
notariales que hoy se presentan en papel al RPI de Neuquén. En lugar de que el
escribano imprima el testimonio en papel, lo firme con su firma ológrafa, lo
acompañe con una minuta y lo presente físicamente por Mesa de Entradas, el
testimonio viaja desde el sistema del Colegio al RPI **completamente por API**,
firmado digitalmente.

Un testimonio digital es:

- Un **XML estructurado** con los datos de uno o más actos (escribano, partes,
  inmueble, monto, etc.), firmado por el escribano autorizante con su
  certificado digital.
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

## Alcance de la v2.0

El cambio central de la v2.0 respecto de la v1.0 es que **un testimonio puede
contener N actos (1 a N)** en lugar de uno solo. Una misma escritura suele
formalizar varios actos (por ejemplo, dos compraventas), y v2 lo modela
explícitamente con una lista `<Actos>` de elementos `<Acto>`.

En cuanto a tipos de acto, la v2.0 sigue cubriendo **un solo tipo**:

- **Compraventa** de inmuebles, escritura notarial.

La estructura ya admite actos de **tipos heterogéneos** dentro de un mismo
testimonio: agregar otros actos (hipoteca, donación, división de condominio,
etc.) es agregar archivos XSD bajo `xsd/v2/actos/` y una línea al `choice` de
`Acto`, sin cambiar la estructura general del XML.

Esta versión soporta, igual que v1:

- **Personas jurídicas** (sociedades, asociaciones) y **organismos públicos**
  como partes del acto, además de personas humanas.
- **Representantes** (tutor, apoderado, etc.) mediante un bloque opcional dentro
  de cada persona.

Y agrega como novedad de v2:

- **N actos por testimonio**, cada uno con sus propias partes, inmuebles, datos
  económicos, certificaciones y visado. Los bloques que en v1 vivían a nivel
  testimonio **bajan al nivel de cada acto**.
- **Partes con rol genérico**: en lugar de los contenedores `Adquirentes` /
  `Transmitentes` de v1, cada acto tiene una lista de `<Parte rol="...">`. Los
  roles definidos hoy son `ADQUIRENTE`, `TRANSMITENTE`, `ACREEDOR` y `DEUDOR`.

No están soportadas en v2.0:

- **Asentimiento conyugal estructurado**: se modela como **texto libre** en el
  campo `AsentimientoConyugal`, no como bloque con cónyuge, fecha, etc.
- **Otros tipos de actos** (hipoteca, donación, división de condominio, etc.):
  se incorporarán en versiones futuras (la estructura ya está preparada).
- **Ampliatorios** o reingresos del mismo testimonio (cada envío genera un
  trámite nuevo).

> v2 **coexiste** con v1: los XSD de v1 quedan intactos en `xsd/` y los de v2
> viven en `xsd/v2/` con su propio namespace (`/v2`). Esta documentación
> describe la v2.0.

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
