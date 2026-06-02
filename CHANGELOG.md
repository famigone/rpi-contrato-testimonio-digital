# Changelog

Todas las modificaciones notables a este contrato se documentan acá.

El formato sigue [Keep a Changelog 1.1.0](https://keepachangelog.com/es-ES/1.1.0/)
y este contrato adhiere a [Semantic Versioning 2.0.0](https://semver.org/spec/v2.0.0.html).

---

## [No publicado]

Cambios pendientes antes de la primera publicación oficial:

### A definir antes de v1.0.0

- URL final del endpoint en staging y producción.
- Mecanismo de autenticación entre cliente y RPI (Bearer token vs mTLS).
- Mecanismo de autenticación de los callbacks (RPI → cliente).
- Requisitos específicos de la cadena de certificación para firma XML-DSig
  en el ámbito del Poder Judicial de Neuquén.
- Email de contacto para reportes de seguridad.

---

## [1.0.0-draft] — 2026-06-02

Borrador inicial del contrato. Se publica para revisión por parte del Colegio
de Escribanos de Neuquén y validación interna del RPI.

### Agregado

#### Contrato técnico

- Endpoint `POST /testimonios` con `multipart/form-data` (partes `xml` y `pdf`).
- Estructura XML del testimonio digital con secciones:
  - `MetadatosEnvio` (identificador, timestamp, hash PDF, versión).
  - `EscribanoAutorizante` (nombre, CUIT, registro, sede).
  - `Otorgamiento` (lugar, fecha, número de escritura, folio).
  - `CertificacionRegistralPrevia` (número, fechas).
  - `DatosEconomicos` (valuación fiscal, monto, moneda, cotización).
  - `Acto` con `Compraventa` (adquirentes, transmitentes, inmuebles).
  - `Rogante`.
  - `Observaciones` (opcional).
  - Firma XML-DSig embebida.

#### Firma digital

- Estándar XML-DSig enveloped con RSA-SHA256 y Canonical XML 1.0.
- Certificado X.509 incluido en `<KeyInfo>`.
- Verificación de coincidencia entre CUIT del certificado y del XML.

#### Adjunto PDF

- PDF firmado digitalmente (recomendado: PAdES).
- Verificación de integridad por hash SHA-256 declarado en el XML.

#### XSDs modulares

- `testimonio-digital.xsd` como entry point.
- 8 archivos comunes reutilizables entre actos en `xsd/comunes/`.
- `xsd/actos/compraventa.xsd` con el único acto soportado en v1.0.
- Inclusión local del XSD oficial de W3C XML-DSig para validación offline.

#### Documentación

- 9 documentos en `docs/` cubriendo introducción, flujo end-to-end, endpoint,
  XML, firma, PDF, errores, callbacks y glosario.
- README principal con índice navegable.
- 3 ejemplos XML válidos en `ejemplos/`.

#### Manejo de errores y reintentos

- Catálogo de códigos HTTP esperados (200, 202, 4xx, 5xx).
- Catálogo de códigos de error específicos del contrato.
- Política de reintentos con backoff exponencial para errores 5xx.
- Garantía de idempotencia por `IdentificadorEnvio` (UUID v4).

#### Callbacks

- Sistema de notificaciones del RPI al cliente con 7 tipos de eventos:
  - `validacion_completada`
  - `sincronizacion_completada`
  - `entrada_general_asignada`
  - `inscripcion_provisoria`
  - `inscripcion_definitiva`
  - `rechazo_registral`
  - `validacion_fallida`

#### Gobernanza

- `GOVERNANCE.md` con roles, procesos de cambio, política de versionado.
- `CONTRIBUTING.md` con guía de contribución.
- `SECURITY.md` con política de reporte de vulnerabilidades.
- `LICENSE.md` con licencia Creative Commons BY 4.0.

### Limitaciones conocidas de v1.0

- Solo se soporta el acto de **compraventa**.
- Solo se aceptan **personas humanas** como adquirentes y transmitentes.
- Sin representantes legales ni poderes.
- Sin asentimiento conyugal estructurado (puede aparecer en el cuerpo del
  testimonio en texto libre).
- Sin manejo avanzado de PEP (solo flag booleano).
- Sin múltiples actos en un mismo testimonio.

Estas limitaciones se abordarán en versiones futuras según las prioridades
del proyecto.

---

## Política de versionado

| Tipo | Formato | Cuándo se aplica | Ejemplo |
|------|---------|------------------|---------|
| MAJOR | `X.0.0` | Cambios incompatibles | Cambiar estructura raíz del XML |
| MINOR | `1.X.0` | Cambios compatibles que agregan funcionalidad | Agregar tipo de acto, campo opcional |
| PATCH | `1.0.X` | Correcciones que no cambian el contrato | Typos, ejemplos, aclaraciones de docs |

El namespace del XML incluye solo la versión MAJOR (`/v1`, `/v2`). Los
cambios MINOR y PATCH mantienen el mismo namespace. Los cambios MAJOR
cambian el namespace.

Ver [GOVERNANCE.md](GOVERNANCE.md) para el proceso completo de releases.
