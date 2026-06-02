# Contrato Testimonio Digital — RPI Neuquén

Especificación técnica del contrato de integración para el envío de **testimonios
digitales** de actos notariales al Registro de la Propiedad Inmueble (RPI) de
la Provincia del Neuquén.

Este repositorio es la **fuente única de verdad** del contrato. Cualquier
cliente que integre con el RPI para enviar testimonios digitales debe ajustarse
a lo definido acá.

---

## Versión actual

| Campo | Valor |
|-------|-------|
| Versión del contrato | 1.0.0 (borrador para revisión) |
| Estado | En proceso de homologación |
| Actos soportados | Compraventa |
| Última actualización | Ver [CHANGELOG.md](CHANGELOG.md) |

Para versiones anteriores, consultar los [tags del repositorio](#versionado).

---

## ¿Quién debería leer esto?

| Audiencia | Por dónde empezar |
|-----------|-------------------|
| Programador que va a integrar un sistema cliente con el RPI | [docs/01-introduccion.md](docs/01-introduccion.md) y luego en orden. |
| Equipo técnico del RPI que mantiene este contrato | [GOVERNANCE.md](GOVERNANCE.md) y [CONTRIBUTING.md](CONTRIBUTING.md). |
| Auditor o revisor que evalúa el contrato | Empezar por [docs/02-flujo-end-to-end.md](docs/02-flujo-end-to-end.md). |
| Consumidores futuros (otros registros, otras integraciones) | Este README + [docs/01-introduccion.md](docs/01-introduccion.md). |

---

## Cómo leer la documentación

Orden recomendado para integraciones nuevas:

1. **[Introducción](docs/01-introduccion.md)** — qué es el testimonio digital y qué problema resuelve.
2. **[Flujo end-to-end](docs/02-flujo-end-to-end.md)** — el ida y vuelta entre cliente y RPI.
3. **[Endpoint API](docs/03-endpoint-api.md)** — URL, método, autenticación, formato de la petición.
4. **[Formato XML](docs/04-formato-xml.md)** — estructura general del XML del testimonio.
5. **[Firma digital](docs/05-firma-digital.md)** — cómo firmar el XML con XML-DSig.
6. **[Adjunto PDF](docs/06-adjunto-pdf.md)** — cómo viaja el PDF junto al XML.
7. **[Respuestas y errores](docs/07-respuestas-y-errores.md)** — códigos HTTP, reintentos.
8. **[Notificaciones de callback](docs/08-notificaciones-callback.md)** — qué te devuelve el RPI cuando inscribe.
9. **[Glosario](docs/09-glosario.md)** — términos del dominio registral y notarial.

---

## Contenido del repositorio

```
.
├── README.md                              ← estás acá
├── CHANGELOG.md                           ← historial de versiones del contrato
├── GOVERNANCE.md                          ← cómo se gobierna este contrato
├── CONTRIBUTING.md                        ← cómo proponer cambios
├── SECURITY.md                            ← reportar vulnerabilidades o problemas
├── LICENSE.md                             ← licencia de uso
│
├── docs/                                  ← documentación funcional y técnica
│   ├── 01-introduccion.md
│   ├── 02-flujo-end-to-end.md
│   ├── 03-endpoint-api.md
│   ├── 04-formato-xml.md
│   ├── 05-firma-digital.md
│   ├── 06-adjunto-pdf.md
│   ├── 07-respuestas-y-errores.md
│   ├── 08-notificaciones-callback.md
│   └── 09-glosario.md
│
├── xsd/                                   ← contratos XSD modulares
│   ├── README.md
│   ├── testimonio-digital.xsd             ← entry point del XSD
│   ├── xmldsig-core-schema.xsd            ← W3C XML-DSig local
│   ├── comunes/
│   │   ├── metadatos-envio.xsd
│   │   ├── escribano-autorizante.xsd
│   │   ├── persona-humana.xsd
│   │   ├── identificacion-inmueble.xsd
│   │   ├── datos-economicos.xsd
│   │   ├── certificacion-registral.xsd
│   │   ├── otorgamiento.xsd
│   │   └── rogante.xsd
│   └── actos/
│       └── compraventa.xsd
│
└── ejemplos/                              ← XMLs válidos de ejemplo
    ├── README.md
    ├── compraventa-minima.xml
    ├── compraventa-multiple-titulares.xml
    └── compraventa-usd.xml
```

---

## Resumen para empezar a programar

Para quien tiene urgencia y quiere ir directo:

- **Endpoint**: `POST` a la URL del RPI (ver [docs/03-endpoint-api.md](docs/03-endpoint-api.md)).
- **Formato**: `multipart/form-data` con dos partes — `xml` (el testimonio firmado) y `pdf` (el documento PDF firmado).
- **Validación**: el XML debe validar contra `xsd/testimonio-digital.xsd`.
- **Firma**: el XML debe estar firmado con XML-DSig por el certificado del escribano autorizante.
- **Respuesta**: HTTP 202 con un `identificadorEnvio` (UUID) para trazabilidad.
- **Notificaciones**: el RPI te notifica los cambios de estado del testimonio vía callback HTTP.

---

## Validación rápida del XSD

Para validar un XML de testimonio contra el contrato:

```bash
xmllint --schema xsd/testimonio-digital.xsd ejemplos/compraventa-minima.xml --noout
```

Los tres ejemplos en `ejemplos/` validan correctamente contra el XSD.

---

## Versionado

Este contrato sigue [Semantic Versioning 2.0.0](https://semver.org/):

| Tipo de cambio | Cuándo | Impacto |
|----------------|--------|---------|
| **MAJOR** (`X.0.0`) | Cambio incompatible | Rompe clientes existentes. Requiere coordinación previa con consumidores. Período de coexistencia. |
| **MINOR** (`1.X.0`) | Funcionalidad nueva compatible | Agregar un acto nuevo, agregar un campo opcional. Los clientes viejos siguen funcionando. |
| **PATCH** (`1.0.X`) | Correcciones que no cambian el contrato | Aclaraciones de documentación, ejemplos nuevos, fixes de typos. |

El namespace XML incluye solo la versión MAJOR (`/v1`, `/v2`). Cambios MINOR
y PATCH mantienen el mismo namespace. Cambios MAJOR cambian el namespace.

Para ver una versión específica:

```bash
git checkout v1.0.0
```

Todas las versiones publicadas tienen un tag Git y aparecen en
[Releases](../../releases).

---

## Cómo proponer un cambio

Si encontrás un problema, ambigüedad, o tenés una propuesta de mejora:

1. **Para preguntas o aclaraciones**: abrir un issue en este repositorio.
2. **Para propuestas de cambio**: leer [CONTRIBUTING.md](CONTRIBUTING.md) antes de abrir un PR.
3. **Para problemas de seguridad**: NO abrir un issue público. Seguir [SECURITY.md](SECURITY.md).

---

## Gobernanza

El proceso de toma de decisiones, los roles, y las reglas para cambios
breaking están en [GOVERNANCE.md](GOVERNANCE.md).

---

## Decisiones pendientes

Antes de salir de "borrador para revisión":

- [ ] Confirmar URL del endpoint en ambientes de staging y producción.
- [ ] Confirmar mecanismo de autenticación (Bearer token vs mTLS).
- [ ] Confirmar requisitos específicos de la cadena de certificación para
      firma XML-DSig en el ámbito del Poder Judicial de Neuquén.

Estos puntos están marcados con `[POR-DEFINIR]` o `⚠️ Pendiente de definición`
en los documentos correspondientes.

---

## Contacto

- **Mantenedor del contrato**: Equipo Técnico del RPI Neuquén.
- **Para consultas técnicas**: abrir issue en este repositorio.
- **Para vulnerabilidades**: ver [SECURITY.md](SECURITY.md).

---

## Licencia

Ver [LICENSE.md](LICENSE.md).
