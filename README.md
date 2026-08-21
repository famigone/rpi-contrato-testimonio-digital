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
| Versión del contrato | 3.0.0 (borrador para revisión) |
| Estado | En proceso de homologación |
| XSD vigente | [`xsd/v3/testimonio-digital.xsd`](xsd/v3/testimonio-digital.xsd), namespace `.../testimonio-digital/v3` |
| Modelo de acto | **El tipo de acto es un DATO** (`<Codigo>`), no un elemento del esquema |
| Actos | Cualquier código del catálogo `act`; qué códigos están **habilitados** lo decide el servicio. Ver [catalogo-actos.json](catalogo-actos.json) |
| Última actualización | Ver [CHANGELOG.md](CHANGELOG.md) |

**El código de acto es un dato.** Un acto se identifica con `<Codigo>NNNN</Codigo>` (el código
del catálogo `act`), no con un elemento nombrado por tipo. El XSD **no** lleva un enum de códigos
(el catálogo cambia y no debe versionar el contrato): la lista de códigos existentes se publica
como dato en [`catalogo-actos.json`](catalogo-actos.json), y qué códigos están *habilitados* lo
valida el servicio. Ver el ADR-004 del repo del servicio.

Las reglas de qué campos y qué partes exige cada acto habilitado se publican en
[`artefacto-campos-por-acto.json`](artefacto-campos-por-acto.json), para que el editor del
escribano arme el formulario dinámicamente. Ver
[docs/11-artefacto-campos-por-acto.md](docs/11-artefacto-campos-por-acto.md).

---

## ¿Quién debería leer esto?

| Audiencia | Por dónde empezar |
|-----------|-------------------|
| Programador que va a integrar un sistema cliente con el RPI | [docs/01-introduccion.md](docs/01-introduccion.md) y luego en orden. |
| Programador del editor del Colegio de Escribanos (formulario del escribano) | [docs/11-artefacto-campos-por-acto.md](docs/11-artefacto-campos-por-acto.md), junto con [docs/10-campos-del-formulario.md](docs/10-campos-del-formulario.md). |
| Equipo técnico del RPI que mantiene este contrato | [GOVERNANCE.md](GOVERNANCE.md) y [CONTRIBUTING.md](CONTRIBUTING.md). |
| Auditor o revisor que evalúa el contrato | Empezar por [docs/02-flujo-end-to-end.md](docs/02-flujo-end-to-end.md). |
| Consumidores futuros (otros registros, otras integraciones) | Este README + [docs/01-introduccion.md](docs/01-introduccion.md). |

---

## Cómo leer la documentación

**Índice completo: [docs/README.md](docs/README.md)** — capítulos, recorridos según tu
caso y una tabla de "dónde encontrar cada cosa".

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
10. **[Campos del formulario](docs/10-campos-del-formulario.md)** — tabla plana de todos los campos para armar el formulario del escribano.
11. **[Artefacto de campos por acto](docs/11-artefacto-campos-por-acto.md)** — cómo consumir `artefacto-campos-por-acto.json` para generar el formulario dinámicamente por acto.

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
│   ├── README.md                          ← ★ índice de la documentación
│   ├── 01-introduccion.md
│   ├── 02-flujo-end-to-end.md
│   ├── 03-endpoint-api.md
│   ├── 04-formato-xml.md
│   ├── 05-firma-digital.md
│   ├── 06-adjunto-pdf.md
│   ├── 07-respuestas-y-errores.md
│   ├── 08-notificaciones-callback.md
│   ├── 09-glosario.md
│   ├── 10-campos-del-formulario.md
│   └── 11-artefacto-campos-por-acto.md   ← guía de integración del artefacto (editor del escribano)
│
├── xsd/
│   ├── README.md
│   ├── xmldsig-core-schema.xsd            ← W3C XML-DSig local
│   └── v3/                                ← ★ contrato VIGENTE (v3.0)
│       ├── README.md
│       ├── testimonio-digital.xsd         ← entry point (Acto con <Codigo>)
│       └── comunes/                       ← tipos comunes (SIN actos/: el acto es un dato)
│           ├── metadatos-envio.xsd
│           ├── escribano-autorizante.xsd
│           ├── otorgamiento.xsd
│           ├── persona.xsd
│           ├── parte.xsd                  ← Persona + rol (ADQUIRENTE/TRANSMITENTE/...)
│           ├── certificacion-inhibicion.xsd
│           ├── identificacion-inmueble.xsd
│           ├── certificacion-catastral.xsd
│           ├── nomenclatura-catastral.xsd
│           ├── certificacion-dominio.xsd
│           ├── datos-economicos.xsd
│           ├── visado-rentas.xsd
│           └── rogante.xsd
│
├── catalogo-actos.json                    ← ★ lista informativa de códigos (dato, no esquema)
├── artefacto-campos-por-acto.json         ← ★ reglas de campos/partes por acto habilitado (ver docs/11)
└── ejemplos/
    ├── README.md
    └── v3/                                ← ★ XMLs válidos (validan contra xsd/v3/)
        ├── compraventa-*.xml              ← compraventas (mínima, usd, jurídica, representante, etc.)
        ├── compraventa-dos-actos.xml      ← testimonio con 2 actos
        ├── hipoteca-ejemplo-valido.xml    ← <Codigo>1075</> (ACREEDOR/DEUDOR, MontoHipoteca)
        ├── donacion-ejemplo-valido.xml    ← <Codigo>1056</>
        ├── permuta-ejemplo-valido.xml     ← <Codigo>1102</>, permuta de 2 actos con cruce de partes
        ├── compraventa-con-hipoteca-ejemplo-valido.xml  ← <Codigo>1028</> + ActoSecundario 1075
        └── cancelacion-hipoteca-ejemplo-valido.xml      ← <Codigo>1020</> (familia LIBERA; SIN catastro)
```

> Los directorios de versiones anteriores (`xsd/`, `xsd/v2/`, `ejemplos/`, `ejemplos/v2/`) siguen
> en disco pero están **congelados** y fuera de esta documentación.

---

## Resumen para empezar a programar

Para quien tiene urgencia y quiere ir directo:

- **Endpoint**: `POST` a la URL del RPI (ver [docs/03-endpoint-api.md](docs/03-endpoint-api.md)).
- **Formato**: `multipart/form-data` con dos partes — `xml` (el testimonio firmado) y `pdf` (el documento PDF firmado).
- **Validación**: el XML debe validar contra `xsd/v3/testimonio-digital.xsd`.
- **Firma**: el XML debe estar firmado con XML-DSig por el certificado del escribano autorizante.
- **Respuesta**: HTTP 202 con un `identificadorEnvio` (UUID) para trazabilidad.
- **Notificaciones**: el RPI te notifica los cambios de estado del testimonio vía callback HTTP.

---

## Validación rápida del XSD

Para validar un XML de testimonio contra el contrato:

```bash
xmllint --schema xsd/v3/testimonio-digital.xsd ejemplos/v3/compraventa-minima.xml --noout
```

Los 12 ejemplos de [`ejemplos/v3/`](ejemplos/v3/) validan contra el XSD v3.

---

## Versionado

Este contrato sigue [Semantic Versioning 2.0.0](https://semver.org/):

| Tipo de cambio | Cuándo | Impacto |
|----------------|--------|---------|
| **MAJOR** (`X.0.0`) | Cambio incompatible | Rompe clientes existentes. Requiere coordinación previa con consumidores. Período de coexistencia. |
| **MINOR** (`3.X.0`) | Funcionalidad nueva compatible | Agregar un campo opcional. Los clientes existentes siguen funcionando. Habilitar un acto nuevo **no** cambia el contrato: es un código más en el servicio. |
| **PATCH** (`3.0.X`) | Correcciones que no cambian el contrato | Aclaraciones de documentación, ejemplos nuevos, fixes de typos. |

El namespace XML incluye solo la versión MAJOR (hoy `/v3`). Cambios MINOR
y PATCH mantienen el mismo namespace. Cambios MAJOR cambian el namespace.

Para ver una versión específica:

```bash
git checkout v3.0.0
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
