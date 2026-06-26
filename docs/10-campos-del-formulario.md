# 10 — Campos del formulario

> Documento destinado al programador que implementa el formulario del lado
> del Colegio. Lista todos los campos que el escribano debe completar para
> generar un testimonio digital válido.
>
> La autoridad técnica final es el XSD (`xsd/v2/testimonio-digital.xsd`). Este
> documento sirve como referencia rápida pero ante discrepancia, el XSD manda.
>
> **v2.0**: un testimonio contiene **N actos**. La sección 4 y sus subsecciones
> (4.1 a 4.6: compraventa, partes, inmuebles —con su catastro—, datos
> económicos, certificación de dominio y visado) se completan **una vez por
> cada acto** dentro de `TestimonioDigital/Actos/Acto`. El catastro
> (certificación catastral y nomenclatura) va **por inmueble**, dentro de cada
> `<Inmueble>` (sección 4.3); la **certificación de inhibición** va **por
> transmitente**, dentro de cada `<Parte rol="TRANSMITENTE">` (sección 4.2). Las
> secciones 1 a 3 y 5 a 7 son a nivel testimonio (una vez por envío).

## Cómo leer este documento

Cada sección corresponde a un bloque del XML. Las tablas tienen estas columnas:

- **Campo del formulario**: nombre sugerido para el campo de UI.
- **Camino XML**: dónde se ubica el dato dentro del testimonio.
- **Tipo**: tipo de dato (Texto, Entero, Fecha, Booleano, Enum, Decimal).
- **Longitud**: longitud máxima (o fija) en caracteres, cuando aplica.
- **Obligatorio**: si el campo siempre se exige, o bajo qué condición.
- **Valores / Notas**: enumeraciones, formatos o aclaraciones.
- **Ejemplo**: valor de muestra.

Las restricciones de longitud y formato las valida el XSD. Las reglas
**condicionales** (ej. "obligatorio si Tipo=H") y las **cruzadas** (ej.
"proporciones suman 1") las valida el servicio del RPI, no el XSD; ver
[07 — Respuestas y errores](07-respuestas-y-errores.md).

## 1. Datos del envío

Camino base: `TestimonioDigital/MetadatosEnvio`

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| Identificador de envío | `IdentificadorEnvio` | Texto (UUID v4) | 36 | Sí | UUID v4 canónico, generado por el sistema del Colegio | `550e8400-e29b-41d4-a716-446655440000` |
| Timestamp de envío | `TimestampEnvio` | Fecha-hora ISO 8601 | — | Sí | Con timezone | `2026-06-15T10:23:45-03:00` |
| Hash del PDF | `HashPDF` | Texto (SHA-256 hex) | 64 | Sí | Hexadecimal lowercase | `3a7bd3e2...4dd4f1b` |
| Versión del contrato | `VersionContrato` | Texto | — | Sí | `MAJOR.MINOR` | `2.0` |

> El atributo raíz `version` de `TestimonioDigital` es obligatorio y fijo en `2.0`.

## 2. Escribano autorizante

Camino base: `TestimonioDigital/EscribanoAutorizante`

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| Nombre | `Nombre` | Texto | 1-60 | Sí | Apellido y nombres | `JUAN CARLOS PÉREZ` |
| CUIT | `CUIT` | Texto | — | Sí | Con o sin guiones | `20-12345678-9` |
| Número de registro | `RegistroNumero` | Texto | 1-8 | Sí | | `123` |
| Sede | `Sede` | Texto | 1-40 | Sí | | `NEUQUÉN` |

## 3. Otorgamiento

Camino base: `TestimonioDigital/Otorgamiento`

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| Lugar | `Lugar` | Texto | 1-60 | Sí | | `NEUQUÉN` |
| Fecha | `Fecha` | Fecha | — | Sí | No futura (servicio) | `2026-06-10` |
| Número de escritura | `NumeroEscritura` | Texto | 1-8 | Sí | Admite ceros a la izquierda | `00012345` |
| Folio del protocolo | `FolioProtocolo` | Texto | 1-8 | Sí | Admite ceros a la izquierda | `00000123` |

## 4. Actos

Camino base: `TestimonioDigital/Actos`

El testimonio contiene **uno o más actos** (`<Acto>`, sin tope). Cada acto se
completa entero con las secciones 4.1 a 4.7 de abajo. En v2.0 todos los actos
son de tipo `Compraventa`, pero la estructura admite tipos heterogéneos.

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| Número de acto | `Acto/@numero` | Entero positivo | — | Sí | Atributo del `<Acto>`. Único entre actos (lo valida el servicio) | `1` |

> **Las secciones 4.1 a 4.6 se completan dentro de cada `<Acto>`.** Su camino
> base es `TestimonioDigital/Actos/Acto/...`. En v1 muchos de estos bloques
> vivían a nivel testimonio; en v2 bajan al acto.
>
> El orden dentro de `<Acto>` es fijo: `Compraventa`, `Partes`, `Inmuebles`,
> `DatosEconomicos`, `CertificacionDominio`, `VisadoRentas`. Dentro de cada
> `<Inmueble>` el orden es: `IdentificacionInmueble`, `CertificacionCatastral`,
> `NomenclaturaCatastral`. Cada `<Parte rol="TRANSMITENTE">` lleva además su
> `CertificacionInhibicion` como último hijo (ver 4.2).

### 4.1 Compraventa (discriminador del acto)

Camino base: `Actos/Acto/Compraventa`

Primer elemento del acto. Lleva solo los campos propios del tipo compraventa
(todos opcionales). Las partes y los inmuebles **no** van acá: van en `Partes`
e `Inmuebles` (4.2 y 4.3).

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| Descripción de acto incompleto | `DescripcionActoIncompleto` | Texto | 106 | No | Aclaración si no es compraventa estándar | `Venta del 50% indiviso` |
| Reconocimiento de hipoteca / medidas cautelares | `ReconocimientoHipotecaMedidasCautelares` | Texto | 4000 | No | Texto libre |
| Afectaciones al dominio | `AfectacionesAlDominio` | Texto | 4000 | No | Texto libre |
| Asentimiento conyugal | `AsentimientoConyugal` | Texto | 4000 | No | Texto libre |

### 4.2 Partes (1 a N por acto)

Camino base: `Actos/Acto/Partes/Parte`

Lista de personas que intervienen en el acto. Reemplaza los contenedores
`Adquirentes`/`Transmitentes` de v1: el rol se indica con el atributo `rol` de
cada `<Parte>`. Cada `<Parte>` tiene el contenido de una persona más ese
atributo.

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| Rol | `Parte/@rol` | Enum | — | Sí | Atributo. `ADQUIRENTE`/`TRANSMITENTE`/`ACREEDOR`/`DEUDOR` | `ADQUIRENTE` |
| Tipo de persona | `Tipo` | Enum | 1 | Sí | `H` / `J` / `O` | `H` |
| Apellido o denominación | `ApellidoODenominacion` | Texto | 1-180 | Sí | Razón social si J/O | `GONZÁLEZ` |
| Nombres | `Nombres` | Texto | 1-80 | Sí si Tipo=H | No aplica si J/O | `MARÍA LAURA` |
| CUIT | `CUIT` | Texto | — | Sí | Con o sin guiones | `27-28456789-3` |
| Tipo de documento | `TipoDocumento` | Enum | 3 | Sí si Tipo=H | `DNI`/`LE`/`LC`/`PAS` | `DNI` |
| Número de documento | `NumeroDocumento` | Texto | 1-20 | Sí | Si J/O, repite el CUIT sin guiones | `28456789` |
| Estado civil | `EstadoCivil` | Enum | — | Sí si adquirente humano | Ver lista abajo | `SOLTERA` |
| Nupcias | `Nupcias` | Texto | 10 | No | | `PRIMERAS` |
| Cónyuge | `Conyuge` | Texto | 80 | No | | `PÉREZ, JUAN` |
| Nacionalidad | `Nacionalidad` | Texto | 20 | Sí si adquirente humano | | `ARGENTINA` |
| Fecha de nacimiento | `FechaNacimiento` | Fecha | — | Sí si adquirente humano | | `1981-03-15` |
| PEP | `PEP` | Booleano | — | Sí | Persona Expuesta Políticamente | `false` |
| Inscripción organismo/sede | `InscripcionOrganismoSede` | Texto | 80 | Sí si Tipo=J/O | | `IGJ NEUQUÉN` |
| Proporción | `Proporcion` | Texto (fracción) | 20 | Sí si rol=ADQUIRENTE | Formato `N/D` | `1/2` |

**Valores de Estado civil**: `SOLTERO`, `SOLTERA`, `CASADO`, `CASADA`,
`DIVORCIADO`, `DIVORCIADA`, `VIUDO`, `VIUDA`, `CONVIVIENTE`, `OTRO`.

**Reglas según rol** (las valida el servicio, no el XSD):

- Rol `ADQUIRENTE` humano: `EstadoCivil`, `Nacionalidad` y `FechaNacimiento`
  son obligatorios; `Proporcion` aplica.
- Rol `TRANSMITENTE` humano: esos tres campos son opcionales; `Proporcion` no
  aplica.
- El XSD acepta cualquier `rol` en cualquier acto; la correspondencia
  rol/tipo-de-acto y que las proporciones de adquirentes **sumen 1 por acto**
  las valida el servicio.

#### Certificación de inhibición (por transmitente)

Camino base: `Actos/Acto/Partes/Parte/CertificacionInhibicion`

La certificación de inhibición recae sobre **la persona que transmite**, no
sobre el acto: por eso es un dato **de cada transmitente**, no del acto. **Cada
`<Parte rol="TRANSMITENTE">` lleva la suya** (vendedor físico o jurídico),
**obligatoria por transmitente** (regla de servicio). Va como **último hijo** de
la `<Parte>`. Los adquirentes **no** la llevan.

> **Certificado conjunto**: si un único certificado de inhibición cubre a varios
> transmitentes, se repite el **mismo `Numero` y `FechaEmision`** en cada uno. Si
> cada vendedor tiene el suyo, se ponen números/fechas distintos.

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| Número | `CertificacionInhibicion/Numero` | Texto | 1-40 | Sí (por transmitente) | | `INH-2026-000789` |
| Fecha de emisión | `CertificacionInhibicion/FechaEmision` | Fecha | — | Sí (por transmitente) | | `2026-06-02` |

#### Representante (opcional)

Camino base: `Actos/Acto/Partes/Parte/Representante` (bloque opcional)

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| Apellido | `Apellido` | Texto | 1-80 | Sí (si hay bloque) | | `MORALES` |
| Nombres | `Nombres` | Texto | 1-80 | Sí (si hay bloque) | | `GUSTAVO ADRIÁN` |
| CUIT | `CUIT` | Texto | — | Sí (si hay bloque) | | `20-27890123-5` |
| Tipo de documento | `TipoDocumento` | Enum | 3 | Sí (si hay bloque) | `DNI`/`LE`/`LC` (sin PAS) | `DNI` |
| Número de documento | `NumeroDocumento` | Texto | 1-20 | Sí (si hay bloque) | | `27890123` |
| PEP | `PEP` | Booleano | — | Sí (si hay bloque) | | `false` |

### 4.3 Inmuebles (1 a 50 por acto)

Camino base: `Actos/Acto/Inmuebles/Inmueble`

Cada acto tiene uno o más `<Inmueble>`. **Cada `<Inmueble>` lleva su propio
catastro**: además de la identificación (4.3.1), incluye una certificación
catastral (4.3.2) y una nomenclatura catastral (4.3.3), **ambas obligatorias por
inmueble**. Si un acto tiene varios inmuebles, cada uno repite los tres bloques.
El orden dentro del `<Inmueble>` es: `IdentificacionInmueble`,
`CertificacionCatastral`, `NomenclaturaCatastral`.

#### 4.3.1 Identificación del inmueble

Camino base: `Actos/Acto/Inmuebles/Inmueble/IdentificacionInmueble`

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| Matrícula | `Matricula` | Entero | 1-8 díg. | Condicional | Requerida si no hay Tomo/Folio/Finca | `3456` |
| Barra | `Barra` | Texto | 5 | No | Barra catastral | `A` |
| Departamento | `Departamento` | Entero (código) | — | Sí | Código 1-16 (ver tabla) | `5` |
| Tomo | `Tomo` | Texto | 10 | Condicional | Para inmuebles antiguos sin matrícula | `145` |
| Folio | `Folio` | Texto | 10 | Condicional | | `231` |
| Finca | `Finca` | Texto | 10 | Condicional | | `8902` |

> Regla de negocio (servicio): cada inmueble debe tener **`Matricula`** o la
> terna **`Tomo`+`Folio`+`Finca`**.

#### Códigos de departamento

| Código | Nombre | Código | Nombre |
|---|---|---|---|
| 1 | Aluminé | 9 | Loncopué |
| 2 | Añelo | 10 | Los Lagos |
| 3 | Catán Lil | 11 | Minas |
| 4 | Collón Curá | 12 | Ñorquín |
| 5 | Confluencia | 13 | Pehuenches |
| 6 | Chos Malal | 14 | Picunches |
| 7 | Huiliches | 15 | Picún Leufú |
| 8 | Lácar | 16 | Zapala |

#### 4.3.2 Certificación catastral (por inmueble)

Camino base: `Actos/Acto/Inmuebles/Inmueble/CertificacionCatastral` (obligatoria
por inmueble)

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| ¿Emitida? | `Emitido` | Booleano | — | Sí | `true`/`false` | `true` |
| Número | `Numero` | Texto | 40 | Sí si Emitido=true | | `CAT-2026-5678` |
| Código de validación | `CodigoValidacion` | Texto | 40 | Sí si Emitido=true | | `VAL-ABCD-1234` |
| ¿Tiene observaciones? | `TieneObservaciones` | Booleano | — | Solo si Emitido=true | `true`/`false` | `false` |
| Observaciones | `Observaciones` | Texto | 4000 | Sí si TieneObservaciones=true | | `El plano registra...` |

#### 4.3.3 Nomenclatura catastral (por inmueble)

Camino base: `Actos/Acto/Inmuebles/Inmueble/NomenclaturaCatastral` (obligatoria
por inmueble; los 5 campos son obligatorios)

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| Campo 1 | `Campo1` | Texto | 2 (fijo) | Sí | | `09` |
| Campo 2 | `Campo2` | Texto | 2 (fijo) | Sí | | `21` |
| Campo 3 | `Campo3` | Texto | 3 (fijo) | Sí | | `045` |
| Campo 4 | `Campo4` | Texto | 4 (fijo) | Sí | | `0123` |
| Campo 5 | `Campo5` | Texto | 4 (fijo) | Sí | | `0008` |

### 4.4 Datos económicos

Camino base: `Actos/Acto/DatosEconomicos`

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| Valuación fiscal | `ValuacionFiscal/valor` | Decimal | 18 díg., 2 dec. | Sí | En pesos. Punto decimal | `5000000.00` |
| Monto | `Monto/valor` | Decimal | 18 díg., 2 dec. | Sí | | `50000000.00` |
| Moneda | `Monto/moneda` | Enum | — | Sí | `$` / `USD` | `$` |
| Cotización | `Monto/cotizacion` | Decimal | 18 díg., 2 dec. | Sí si moneda=USD | Pesos por unidad | `1100.50` |

### 4.5 Certificación de dominio

Camino base: `Actos/Acto/CertificacionDominio`

Certificado registral de **dominio** (sobre el inmueble), a **nivel acto**. Junto
con la certificación de inhibición (que va por transmitente, ver 4.2), reemplaza
al antiguo bloque `CertificacionRegistralPrevia` (número + dos fechas).
Estructuralmente opcional en el XSD, pero **obligatorio para compraventa** (lo
exige el servicio del RPI).

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| Número | `Numero` | Texto | 1-40 | Sí (compraventa) | | `2026-001234` |
| Fecha de emisión | `FechaEmision` | Fecha | — | Sí (compraventa) | | `2026-06-01` |

### 4.6 Visado de Rentas

Camino base: `Actos/Acto/VisadoRentas` (bloque obligatorio)

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| Tipo de visado | `Tipo` | Enum | — | Sí | `R` (con visado) / `A` (sin visado) | `R` |
| Número de trámite | `NumeroTramite` | Texto | 10 | Sí si Tipo=R | | `RNT-987654` |

## 5. Rogante

Camino base: `TestimonioDigital/Rogante` (nivel testimonio: uno por envío)

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| CUIT | `CUIT` | Texto | — | Sí | | `20-12345678-9` |
| Nombre | `Nombre` | Texto | 1-120 | Sí | | `JUAN CARLOS PÉREZ` |
| Número de registro | `NumeroRegistro` | Texto | 1-20 | Sí | | `123` |
| Localidad | `Localidad` | Texto | 1-30 | Sí | | `NEUQUÉN` |
| Provincia | `Provincia` | Texto | 1-20 | Sí | | `NEUQUÉN` |
| Domicilio | `Domicilio` | Texto | 1-40 | Sí | | `CALLE FALSA 123` |
| Teléfono | `Telefono` | Texto | 1-20 | Sí | | `0299-4567890` |

## 6. Cuerpo de la escritura

Camino base: `TestimonioDigital/TextoCuerpo` (nivel testimonio: uno por envío,
cubre todos los actos). Texto del cuerpo de la escritura que el escribano
transcribe del PDF.

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas |
|---|---|---|---|---|---|
| Cuerpo de la escritura | `TextoCuerpo` | Texto plano | 1 - 500.000 | Sí | Transcripción fiel del cuerpo de la escritura del PDF. Saltos de línea naturales preservados. Sin HTML/Markdown. |

## 7. Observaciones (opcional)

Camino XML: `TestimonioDigital/Observaciones` (nivel testimonio). Texto libre,
hasta 4000 caracteres. Opcional.
