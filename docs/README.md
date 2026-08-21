# Índice de la documentación

Contrato **Testimonio Digital — RPI Neuquén**, versión **3.0** (namespace
`https://contrato.rpi.jusneuquen.gov.ar/testimonio-digital/v3`).

Para el panorama del repositorio (XSD, catálogo, ejemplos, gobernanza), ver el
[README raíz](../README.md).

---

## Por dónde empezar según tu caso

| Si sos… | Leé, en este orden |
|---------|--------------------|
| Programador que integra un sistema cliente con el RPI | [01](01-introduccion.md) → [02](02-flujo-end-to-end.md) → [03](03-endpoint-api.md) → [04](04-formato-xml.md) → [05](05-firma-digital.md) → [06](06-adjunto-pdf.md) → [07](07-respuestas-y-errores.md) → [08](08-notificaciones-callback.md) |
| Programador del editor del escribano (formulario) | [11](11-artefacto-campos-por-acto.md) → [10](10-campos-del-formulario.md) → [04](04-formato-xml.md) |
| Tenés urgencia y querés mandar un testimonio ya | [03](03-endpoint-api.md) + [04](04-formato-xml.md) + [05](05-firma-digital.md), y mirá [`ejemplos/v3/`](../ejemplos/v3/) |
| Auditor o revisor del contrato | [02](02-flujo-end-to-end.md) → [04](04-formato-xml.md) → [05](05-firma-digital.md) |
| Estás perdido con un término del dominio | [09 — Glosario](09-glosario.md) |

---

## Capítulos

| # | Capítulo | Qué contiene |
|---|----------|--------------|
| 01 | [Introducción](01-introduccion.md) | Qué es el testimonio digital, qué problema resuelve, alcance del contrato (N actos, el acto como código, qué no está soportado) y roles de la integración. |
| 02 | [Flujo end-to-end](02-flujo-end-to-end.md) | El ida y vuelta completo entre el Colegio y el RPI: diagrama de secuencia, paso a paso, tiempos esperados, idempotencia. |
| 03 | [Endpoint API](03-endpoint-api.md) | URL, método, autenticación, `multipart/form-data` (partes `xml` y `pdf`), respuestas 202/200, límites técnicos y headers. |
| 04 | [Formato XML](04-formato-xml.md) | La estructura del testimonio: namespace, secciones del XML, `<Acto>` con su `<Codigo>`, partes con rol, inmuebles y certificaciones, validación. |
| 05 | [Firma digital](05-firma-digital.md) | XML-DSig: algoritmos, canonicalización, certificado del firmante, qué verifica el RPI, bibliotecas y errores comunes. |
| 06 | [Adjunto PDF](06-adjunto-pdf.md) | Cómo viaja el PDF, su firma, y el hash SHA-256 que lo liga al XML. |
| 07 | [Respuestas y errores](07-respuestas-y-errores.md) | Códigos HTTP, formato del cuerpo de error, **catálogo completo de códigos** y política de reintentos. |
| 08 | [Notificaciones de callback](08-notificaciones-callback.md) | Cómo el RPI avisa los cambios de estado (provisorio, definitivo, rechazo): endpoint del Colegio, payload, reintentos, seguridad. |
| 09 | [Glosario](09-glosario.md) | Términos registrales, notariales y técnicos del contrato. |
| 10 | [Campos del formulario](10-campos-del-formulario.md) | Tabla plana de **todos** los campos, con su camino XML, tipo, longitud y obligatoriedad. |
| 11 | [Artefacto de campos por acto](11-artefacto-campos-por-acto.md) | Cómo consumir `artefacto-campos-por-acto.json` para **generar** el formulario por acto: partes, campos, asentimiento condicional, proporciones. |

---

## Dónde encontrar cada cosa

| Pregunta | Dónde |
|----------|-------|
| ¿A qué URL mando el testimonio? | [03 — URL del endpoint](03-endpoint-api.md#url-del-endpoint) |
| ¿Cómo me autentico? | [03 — Autenticación](03-endpoint-api.md#autenticación) |
| ¿Cómo se arma el XML? | [04 — Estructura general](04-formato-xml.md#estructura-general) |
| ¿Cómo indico el tipo de acto? | [04 — Codigo (tipo del acto)](04-formato-xml.md#codigo-tipo-del-acto) |
| ¿Qué códigos de acto existen? | [`catalogo-actos.json`](../catalogo-actos.json) |
| ¿Qué códigos puedo mandar hoy? | [11 §1](11-artefacto-campos-por-acto.md#1-panorama-las-tres-piezas-del-repo-del-contrato) y [`artefacto-campos-por-acto.json`](../artefacto-campos-por-acto.json) |
| ¿Qué campos exige cada acto? | [11 §4](11-artefacto-campos-por-acto.md#4-estructura-del-nivel-2-una-entrada-de-acto) |
| ¿Cuándo pido asentimiento conyugal? | [11 §5.1](11-artefacto-campos-por-acto.md#51-asentimiento-conyugal--condicional) |
| ¿Las proporciones tienen que sumar 1? | [11 §5.2](11-artefacto-campos-por-acto.md#52-proporciones--suman-1-o-no) |
| ¿Dónde va cada certificación? | [04 — Secciones del XML](04-formato-xml.md#secciones-del-xml) y [04 — La Parte](04-formato-xml.md#la-parte-persona--rol) |
| ¿Qué longitud tiene tal campo? | [10 — Campos del formulario](10-campos-del-formulario.md) |
| ¿Cómo firmo el XML? | [05 — Estructura de la firma](05-firma-digital.md#estructura-de-la-firma) |
| ¿Cómo calculo el hash del PDF? | [06 — Cómo calcular el hash](06-adjunto-pdf.md#cómo-calcular-el-hash) |
| Me devolvió un error, ¿qué significa? | [07 — Catálogo de códigos de error](07-respuestas-y-errores.md#catálogo-de-códigos-de-error) |
| ¿Reintento o no? | [07 — Política de reintentos](07-respuestas-y-errores.md#política-de-reintentos) |
| ¿Cómo me entero de que se inscribió? | [08 — Tipos de eventos](08-notificaciones-callback.md#tipos-de-eventos) |
| ¿Qué quiere decir "rogante" / "asiento" / "minuta"? | [09 — Glosario](09-glosario.md) |

---

## Más allá de la documentación

| Recurso | Para qué |
|---------|----------|
| [`xsd/v3/testimonio-digital.xsd`](../xsd/v3/testimonio-digital.xsd) | La especificación **precisa**. Ante discrepancia con estos documentos, manda el XSD. |
| [`ejemplos/v3/`](../ejemplos/v3/) | 12 XMLs válidos: compraventa (varias formas), hipoteca, donación, permuta, cancelación, acto secundario. |
| [`catalogo-actos.json`](../catalogo-actos.json) | Los códigos de acto existentes (dato informativo, no esquema). |
| [`artefacto-campos-por-acto.json`](../artefacto-campos-por-acto.json) | Reglas de partes y campos por acto habilitado. |
| [CHANGELOG.md](../CHANGELOG.md) | Qué cambió en cada versión del contrato. |
| [CONTRIBUTING.md](../CONTRIBUTING.md) · [GOVERNANCE.md](../GOVERNANCE.md) | Cómo proponer un cambio y cómo se decide. |

Para validar un XML contra el contrato:

```bash
xmllint --schema xsd/v3/testimonio-digital.xsd ejemplos/v3/compraventa-minima.xml --noout
```
