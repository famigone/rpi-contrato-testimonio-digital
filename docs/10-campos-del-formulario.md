# 10 — Campos del formulario

> Documento destinado al programador que implementa el formulario del lado
> del Colegio. Lista todos los campos que el escribano debe completar para
> generar un testimonio digital válido.
>
> La autoridad técnica final es el XSD (`xsd/testimonio-digital.xsd`). Este
> documento sirve como referencia rápida pero ante discrepancia, el XSD manda.

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
| Versión del contrato | `VersionContrato` | Texto | — | Sí | `MAJOR.MINOR` | `1.0` |

> El atributo raíz `version` de `TestimonioDigital` es obligatorio y fijo en `1.0`.

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

## 4. Certificación registral previa

Camino base: `TestimonioDigital/CertificacionRegistralPrevia`

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| Número | `Numero` | Texto | 1-40 | Sí | | `2026-001234` |
| Fecha emisión (primera) | `FechaEmisionPrimera` | Fecha | — | Sí | | `2026-06-01` |
| Fecha emisión (segunda) | `FechaEmisionSegunda` | Fecha | — | Sí | | `2026-06-30` |

## 5. Certificación catastral

Camino base: `TestimonioDigital/CertificacionCatastral` (bloque obligatorio)

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| ¿Emitida? | `Emitido` | Booleano | — | Sí | `true`/`false` | `true` |
| Número | `Numero` | Texto | 40 | Sí si Emitido=true | | `CAT-2026-5678` |
| Código de validación | `CodigoValidacion` | Texto | 40 | Sí si Emitido=true | | `VAL-ABCD-1234` |
| ¿Tiene observaciones? | `TieneObservaciones` | Booleano | — | Solo si Emitido=true | `true`/`false` | `false` |
| Observaciones | `Observaciones` | Texto | 4000 | Sí si TieneObservaciones=true | | `El plano registra...` |

## 6. Nomenclatura catastral (opcional)

Camino base: `TestimonioDigital/NomenclaturaCatastral` (bloque opcional; si se
incluye, los 5 campos son obligatorios)

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| Campo 1 | `Campo1` | Texto | 2 (fijo) | Sí (si hay bloque) | | `09` |
| Campo 2 | `Campo2` | Texto | 2 (fijo) | Sí (si hay bloque) | | `21` |
| Campo 3 | `Campo3` | Texto | 3 (fijo) | Sí (si hay bloque) | | `045` |
| Campo 4 | `Campo4` | Texto | 4 (fijo) | Sí (si hay bloque) | | `0123` |
| Campo 5 | `Campo5` | Texto | 4 (fijo) | Sí (si hay bloque) | | `0008` |

## 7. Datos económicos

Camino base: `TestimonioDigital/DatosEconomicos`

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| Valuación fiscal | `ValuacionFiscal/valor` | Decimal | 18 díg., 2 dec. | Sí | En pesos. Punto decimal | `5000000.00` |
| Monto | `Monto/valor` | Decimal | 18 díg., 2 dec. | Sí | | `50000000.00` |
| Moneda | `Monto/moneda` | Enum | — | Sí | `$` / `USD` | `$` |
| Cotización | `Monto/cotizacion` | Decimal | 18 díg., 2 dec. | Sí si moneda=USD | Pesos por unidad | `1100.50` |

## 8. Visado de Rentas

Camino base: `TestimonioDigital/VisadoRentas` (bloque obligatorio)

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| Tipo de visado | `Tipo` | Enum | — | Sí | `R` (con visado) / `A` (sin visado) | `R` |
| Número de trámite | `NumeroTramite` | Texto | 10 | Sí si Tipo=R | | `RNT-987654` |

## 9. Acto — Compraventa

Camino base: `TestimonioDigital/Acto/Compraventa`

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| Descripción de acto incompleto | `DescripcionActoIncompleto` | Texto | 106 | No | Aclaración si no es compraventa estándar | `Venta del 50% indiviso` |

### 9.1 Adquirentes (1 a 20 personas)

Camino base: `Acto/Compraventa/Adquirentes/Persona`

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
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
| Inscripción organismo/sede | `InscripcionOrganismoSede` | Texto | 80 | Sí si Tipo=J/O | | `IGJ - CABA` |
| Proporción | `Proporcion` | Texto (fracción) | 20 | Sí | Formato `N/D` | `1/2` |

**Valores de Estado civil**: `SOLTERO`, `SOLTERA`, `CASADO`, `CASADA`,
`DIVORCIADO`, `DIVORCIADA`, `VIUDO`, `VIUDA`, `CONVIVIENTE`, `OTRO`.

#### Representante (opcional)

Camino base: `Adquirentes/Persona/Representante` (bloque opcional)

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| Apellido | `Apellido` | Texto | 1-80 | Sí (si hay bloque) | | `MORALES` |
| Nombres | `Nombres` | Texto | 1-80 | Sí (si hay bloque) | | `GUSTAVO ADRIÁN` |
| CUIT | `CUIT` | Texto | — | Sí (si hay bloque) | | `20-27890123-5` |
| Tipo de documento | `TipoDocumento` | Enum | 3 | Sí (si hay bloque) | `DNI`/`LE`/`LC` (sin PAS) | `DNI` |
| Número de documento | `NumeroDocumento` | Texto | 1-20 | Sí (si hay bloque) | | `27890123` |
| PEP | `PEP` | Booleano | — | Sí (si hay bloque) | | `false` |

### 9.2 Transmitentes (1 a 20 personas)

Camino base: `Acto/Compraventa/Transmitentes/Persona`

Misma estructura que adquirentes, con dos diferencias:

- `EstadoCivil`, `Nacionalidad` y `FechaNacimiento` son **opcionales** para
  transmitentes humanos (para adquirentes humanos son obligatorios).
- `Proporcion` **no aplica** a transmitentes.

El bloque `Representante` también está disponible para transmitentes, con los
mismos 6 campos descritos en 9.1.

### 9.3 Inmuebles (1 a 50)

Camino base: `Acto/Compraventa/Inmuebles/Inmueble/IdentificacionInmueble`

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

### 9.4 Bloques de texto libre (opcionales)

Camino base: `Acto/Compraventa`

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas |
|---|---|---|---|---|---|
| Reconocimiento de hipoteca / medidas cautelares | `ReconocimientoHipotecaMedidasCautelares` | Texto | 4000 | No | Texto libre |
| Afectaciones al dominio | `AfectacionesAlDominio` | Texto | 4000 | No | Texto libre |
| Asentimiento conyugal | `AsentimientoConyugal` | Texto | 4000 | No | Texto libre |

## 10. Rogante

Camino base: `TestimonioDigital/Rogante`

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas | Ejemplo |
|---|---|---|---|---|---|---|
| CUIT | `CUIT` | Texto | — | Sí | | `20-12345678-9` |
| Nombre | `Nombre` | Texto | 1-120 | Sí | | `JUAN CARLOS PÉREZ` |
| Número de registro | `NumeroRegistro` | Texto | 1-20 | Sí | | `123` |
| Localidad | `Localidad` | Texto | 1-30 | Sí | | `NEUQUÉN` |
| Provincia | `Provincia` | Texto | 1-20 | Sí | | `NEUQUÉN` |
| Domicilio | `Domicilio` | Texto | 1-40 | Sí | | `CALLE FALSA 123` |
| Teléfono | `Telefono` | Texto | 1-20 | Sí | | `0299-4567890` |

## 11. Cuerpo del acto

Texto del cuerpo de la escritura que el escribano transcribe del PDF.

| Campo del formulario | Camino XML | Tipo | Longitud | Obligatorio | Valores / Notas |
|---|---|---|---|---|---|
| Cuerpo del acto | `TextoCuerpo` | Texto plano | 1 - 500.000 | Sí | Transcripción fiel del cuerpo de la escritura del PDF. Saltos de línea naturales preservados. Sin HTML/Markdown. |

## 12. Observaciones (opcional)

Camino XML: `TestimonioDigital/Observaciones`. Texto libre, hasta 4000
caracteres. Opcional.
