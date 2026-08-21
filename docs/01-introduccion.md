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

## Alcance del contrato

**Un testimonio contiene N actos (1 a N).** Una misma escritura suele formalizar
varios actos (por ejemplo, dos compraventas), y el contrato lo modela con una
lista `<Actos>` de elementos `<Acto>`.

**El tipo de acto es un CÓDIGO, no un elemento del esquema.** Un acto se
identifica con `<Codigo>NNNN</Codigo>` (el código del catálogo `act`). El XSD no
lleva un enum de códigos: la lista de códigos existentes se publica como dato en
[`catalogo-actos.json`](../catalogo-actos.json), y qué códigos están
*habilitados* lo decide el servicio. Ver el ADR-004 del repo del servicio.

Como consecuencia, **agregar un acto no toca el contrato**: es habilitar un
código más en el servicio (con su familia estructural), no agregar un XSD ni una
rama del esquema. La variabilidad por acto (qué roles, qué montos, qué
certificaciones exige) se valida en el servicio por combinación código→familia,
no en el XSD. Actos con evidencia hasta ahora: compraventa (1028), hipoteca
(1075), donación (1056), permuta (1102), y la familia de
cancelaciones/liberaciones (1020, 1088, …).

El contrato soporta:

- **Personas jurídicas** (sociedades, asociaciones) y **organismos públicos**
  como partes del acto, además de personas humanas.
- **Representantes** (tutor, apoderado, etc.) mediante un bloque opcional dentro
  de cada persona.
- **N actos por testimonio**, cada uno con sus propias partes, inmuebles, datos
  económicos, certificaciones y visado.
- **Partes con rol genérico**: cada acto tiene una lista de `<Parte rol="...">`.
  Los roles definidos hoy son `ADQUIRENTE`, `TRANSMITENTE`, `ACREEDOR` y
  `DEUDOR`.
- **Actos sin catastro** (por ejemplo, cancelación de hipoteca): la certificación
  catastral y la nomenclatura son opcionales.

No están soportados:

- **Asentimiento conyugal estructurado**: se modela como **texto libre** en el
  campo `AsentimientoConyugal` (hijo opcional de `<Acto>`), no como bloque con
  cónyuge, fecha, etc.
- **Actos no habilitados**: el catálogo tiene 231 códigos, pero solo los que el
  servicio habilita pueden enviarse por testimonio digital. Ver
  [11 — Artefacto de campos por acto](11-artefacto-campos-por-acto.md).
- **Ampliatorios** o reingresos del mismo testimonio (cada envío genera un
  trámite nuevo).

> Esta documentación describe la **v3.0**, la versión vigente del contrato. El
> XSD es [`xsd/v3/testimonio-digital.xsd`](../xsd/v3/testimonio-digital.xsd),
> con namespace `https://contrato.rpi.jusneuquen.gov.ar/testimonio-digital/v3`.

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

Para ver todos los capítulos y buscar un tema puntual, mirá el
[índice de la documentación](README.md).
