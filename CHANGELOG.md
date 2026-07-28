# Changelog

Todas las modificaciones notables a este contrato se documentan acá.

El formato sigue [Keep a Changelog 1.1.0](https://keepachangelog.com/es-ES/1.1.0/)
y este contrato adhiere a [Semantic Versioning 2.0.0](https://semver.org/spec/v2.0.0.html).

---

## [3.0.0] — 2026-07-24

Cambio **MAJOR (incompatible)**: el **tipo de acto pasa de ser un elemento del esquema a ser un
DATO**. En v2 cada tipo era un elemento del `xs:choice` de `ActoType` (`<Compraventa/>`,
`<Hipoteca/>`, `<Donacion/>`, `<Permuta/>`) con un `xs:complexType` propio; esos tipos estaban
**vacíos** (solo textos libres). v3 los elimina y modela el acto con **`<Codigo>`** (el código del
catálogo legacy `act`). Namespace nuevo `.../testimonio-digital/v3`, `version="3.0"`. Ver
[ADR-004](https://github.com/) (repo del servicio, `docs/adr/004-contrato-v3-codigo-como-dato.md`).

**Por qué ahora:** hay 4 actos y **cero emisores en producción real** (las compraventas
sincronizadas fueron en *staging*). El breaking change es barato hoy; en seis meses, con emisores
integrados, sería caro.

### ⚠️ Cómo migrar (para sistemas notariales que consumen el contrato)

- **El elemento de acto se reemplaza por `<Codigo>`**, primer hijo del `<Acto>`:
  - `<Compraventa/>` → `<Codigo>1028</Codigo>`
  - `<Hipoteca/>` → `<Codigo>1075</Codigo>`
  - `<Donacion/>` → `<Codigo>1056</Codigo>`
  - `<Permuta/>` → `<Codigo>1102</Codigo>`
- **Los textos libres** que vivían dentro del elemento de acto (`DescripcionActoIncompleto`,
  `ReconocimientoHipotecaMedidasCautelares`, `AfectacionesAlDominio`, `AsentimientoConyugal`)
  ahora son **hijos opcionales de `<Acto>`**, al final (después de `VisadoRentas`).
- **`<ActoSecundario>`** (código, desde v2.4.0) **no cambia**: va después de `<Codigo>`.
- **Namespace y versión**: `xmlns=".../v3"`, `version="3.0"`, `<VersionContrato>3.0</VersionContrato>`,
  `schemaLocation` → `xsd/v3/testimonio-digital.xsd`.

### Agregado
- **`xsd/v3/`**: el contrato v3 completo. `ActoType` con `<Codigo>` (`CodigoActoType`, `xs:integer`
  ≤ 9999 — **NO** un enum de los ~200 códigos, ver abajo) + los textos libres opcionales.
- **`catalogo-actos.json`** (raíz del repo): lista **informativa** de códigos existentes (código +
  nombre). Es **dato, no esquema**: NO valida nada. Compensa la ausencia del enum. Qué códigos están
  *habilitados* lo decide el servicio, no este archivo.
- **`ejemplos/v3/`**: los 11 ejemplos de v2 migrados + **`cancelacion-hipoteca-ejemplo-valido.xml`**
  (código 1020) — un caso que **v2 NO podía expresar** (el catastro era obligatorio) y v3 habilita.

### Cambiado (incompatible)
- **`<Codigo>` es `xs:integer`, no un enum.** Razón: el catálogo legacy cambia (5 altas en 7 meses:
  1326/1327 dic-2025, 1330 mar-2026, 1334/1335 jun-2026). Con enum, cada alta del catálogo sería un
  release del contrato aunque el servicio no habilite ese acto. El contrato no debe versionarse por
  cambios del catálogo. La existencia de códigos se documenta en `catalogo-actos.json`.
- **`CertificacionCatastral` y `NomenclaturaCatastral` pasan a `minOccurs=0`** (eran obligatorias por
  inmueble). La familia HIPOTECARIA-LIBERA (cancelación, liberación) no las lleva; su obligatoriedad
  para las familias que sí las requieren se valida en el servicio (regla `exigeCatastral`). Es la
  **única** obligatoriedad estructural que v3 relaja.

### Eliminado
- **`xsd/v3/actos/`** no existe: los 4 XSD de acto de v2 (`compraventa.xsd`, `hipoteca.xsd`,
  `donacion.xsd`, `permuta.xsd`) no tienen equivalente — el acto ya no tiene tipo XSD propio.

### Sin cambios
- Todo lo demás de v2: modelo de partes con rol, N actos, Inmuebles/DatosEconomicos/VisadoRentas
  obligatorios, acto secundario, componentes comunes (`comunes/`), firma XML-DSig.
- **v2 queda congelado en `xsd/v2/`** (coexiste, como v1 en su momento).

---

## [2.4.0] — 2026-07-23

Cambio **MINOR** (aditivo y retrocompatible): se incorpora el **ACTO SECUNDARIO** de una
minuta. Es una **dimensión del modelo** que faltaba, no un acto nuevo. Mismo namespace y
`version="2.0"`; todos los testimonios existentes siguen validando (el elemento es opcional).

El legacy modela "una minuta = acto principal + acto secundario" (columnas `wmmac1`/`wmmgr1` +
`wmmac2`/`wmmgr2`). El caso frecuente es la **compraventa con hipoteca simultánea** (comprar con
crédito): UN solo acto registral con DOS códigos. Hasta 2.3.0 el contrato no podía expresarlo (un
`<Acto>` lleva un solo tipo por el `xs:choice`), así que ese caso —el más frecuente del registro—
no tenía canal correcto.

### Agregado
- `testimonio-digital.xsd`: elemento **opcional** `<ActoSecundario>` dentro de `ActoType`
  (inmediatamente después del choice del acto principal). Su tipo `CodigoActoSecundarioEnum` está
  **restringido a los dos códigos** que el legacy admite como secundario: **1075** (HIPOTECA) y
  **1157** (RECONOCIMIENTO DE HIPOTECA).
- Es la **primera vez** que el contrato usa un **código** en vez de un nombre de elemento. Está
  justificado: el acto secundario **no tiene estructura propia** (partes, inmuebles y montos son
  compartidos por ambos actos del mismo `<Acto>`), es un **marcador** del segundo código. Anticipa
  la dirección del v3 (actos por código de catálogo). Documentado así en el XSD.
- Ejemplo `ejemplos/v2/compraventa-con-hipoteca-ejemplo-valido.xml` (datos **ficticios**):
  compraventa (principal) + `ActoSecundario=1075`, con ADQUIRENTE + TRANSMITENTE + ACREEDOR y los
  **tres montos** en el mismo acto (precio + valuación + `MontoHipoteca`). El vendedor y el
  acreedor son la misma entidad (I.P.V.U.) con el mismo CUIT en dos roles.

### Sin cambios
- El `xs:choice` del acto **principal** no cambia (Compraventa/Hipoteca/Donacion/Permuta).
- `comunes/datos-economicos.xsd` **no se toca**: los tres bloques económicos (`ValuacionFiscal`,
  `Monto`, `MontoHipoteca`) ya coexistían; una compraventa-con-hipoteca los usa a los tres.
- Qué acto principal admite secundario, y las reglas que el secundario dispara (1075 exige
  `MontoHipoteca` y rol `ACREEDOR`), se validan por **reglas de negocio del servicio**, no por el
  XSD.

---

## [2.3.0] — 2026-07-21

Cambio **MINOR** (aditivo y retrocompatible): se incorpora el acto de **Permuta**.
Mismo namespace y `version="2.0"`; compraventa, hipoteca y donación existentes siguen
validando sin cambios. Permuta es **estructuralmente idéntica a compraventa** (partes
con rol `TRANSMITENTE` y `ADQUIRENTE`; inmuebles y montos a nivel `<Acto>`) y **reutiliza
el bloque económico** de compraventa (`ValuacionFiscal` + `Monto`): no agrega estructura
económica propia.

Modelo de la permuta: una permuta de N inmuebles se representa como **UN testimonio con
N `<Acto>` de permuta** (uno por matrícula), agrupados por el mismo trámite y un solo PDF
firmado. Cada acto es **simple** —un transmitente y un adquirente—; el "cruce" característico
(una persona transmite en un acto y adquiere en otro) ocurre **entre actos**, nunca dentro
de uno. El **PRECIO** (`Monto`) se **repite igual en cada acto** —es el valor de la operación,
no se reparte—; la **VALUACIÓN FISCAL** (`ValuacionFiscal`) es la del inmueble de cada acto y
por eso difiere entre actos.

### Agregado
- `xsd/v2/actos/permuta.xsd`: tipo `PermutaType`. Mismo patrón que `CompraventaType`
  (textos libres opcionales del acto, incluido `AsentimientoConyugal`); el sustento vive
  a nivel `<Acto>`.
- `<xs:element name="Permuta" type="PermutaType"/>` **activado** en el `xs:choice` de
  `ActoType` (`testimonio-digital.xsd`) + include del nuevo XSD.
- Ejemplo `ejemplos/v2/permuta-ejemplo-valido.xml` (datos **ficticios**) con **dos actos**
  de permuta: el mismo `Monto` repetido en ambos y valuaciones fiscales distintas.

### Sin cambios
- `comunes/datos-economicos.xsd` **no se toca**: permuta usa `ValuacionFiscal`/`Monto`
  tal como están (opcionales a nivel XSD). Qué monto exige cada acto lo validan las reglas
  de negocio del servicio: permuta exige `ValuacionFiscal` + `Monto` (precio), igual que
  compraventa.
- Los roles `ADQUIRENTE`/`TRANSMITENTE` y su ruteo ya existían: permuta no agrega roles.

---

## [2.2.0] — 2026-07-16

Cambio **MINOR** (aditivo y retrocompatible): se incorpora el acto de **Donación**.
Mismo namespace y `version="2.0"`; compraventa e hipoteca existentes siguen validando
sin cambios. Donación es **estructuralmente idéntica a compraventa** (partes DONANTE =
`TRANSMITENTE`, DONATARIO/A = `ADQUIRENTE`; inmuebles y montos a nivel `<Acto>`) y
**reutiliza el bloque económico** de compraventa (el valor del acto va en
`DatosEconomicos/Monto`): no agrega estructura económica propia.

### Agregado
- `xsd/v2/actos/donacion.xsd`: tipo `DonacionType`. Mismo patrón que `CompraventaType`
  (textos libres opcionales del acto, incluido `AsentimientoConyugal`); el sustento vive
  a nivel `<Acto>`.
- `<xs:element name="Donacion" type="DonacionType"/>` **activado** en el `xs:choice` de
  `ActoType` (`testimonio-digital.xsd`) + include del nuevo XSD.
- Ejemplo `ejemplos/v2/donacion-ejemplo-valido.xml` (datos **ficticios**).

### Sin cambios
- `comunes/datos-economicos.xsd` **no se toca**: donación usa `ValuacionFiscal`/`Monto`
  tal como quedaron en 2.1.0 (opcionales). El `Monto` es el valor del acto de donación.

---

## [2.1.0] — 2026-07-14

Cambio **MINOR** (aditivo y retrocompatible): se incorpora el acto de **Hipoteca**.
Mismo namespace y `version="2.0"`; los testimonios de compraventa existentes siguen
validando sin cambios. Los roles `ACREEDOR`/`DEUDOR` (ya presentes desde 2.0.0) pasan
a estar **disponibles** con este release.

### Agregado
- `xsd/v2/actos/hipoteca.xsd`: tipo `HipotecaType`. Es el discriminador del acto en el
  `xs:choice`; el sustento (partes con rol `ACREEDOR`/`DEUDOR`, inmuebles, montos) vive a
  nivel `<Acto>`, igual que en compraventa. Sin campos propios por ahora.
- `<xs:element name="Hipoteca" type="HipotecaType"/>` **activado** en el `xs:choice` de
  `ActoType` (`testimonio-digital.xsd`) + include del nuevo XSD.
- `comunes/datos-economicos.xsd`: elemento `MontoHipoteca` (capital del gravamen:
  `valor` + `moneda` + `cotizacion`), del mismo tipo que `Monto`.
- Ejemplo `ejemplos/v2/hipoteca-ejemplo-valido.xml` (datos **ficticios**).

### Cambiado
- `comunes/datos-economicos.xsd`: `ValuacionFiscal` y `Monto` pasan a `minOccurs=0`
  (opcionales a nivel XSD). Es una **relajación retrocompatible**: los testimonios de
  compraventa que ya los incluyen siguen validando. Qué monto exige cada acto se valida por
  **reglas de negocio del servicio**, no por el XSD: compraventa exige `ValuacionFiscal` +
  `Monto` (precio); hipoteca exige `MontoHipoteca` y **no** lleva valuación ni precio de venta.

---

## [2.0.0] — 2026-06-24

Cambio **MAJOR**: un testimonio pasa de **un acto** a **N actos (1..N)**. Nuevo
namespace `https://contrato.rpi.jusneuquen.gov.ar/testimonio-digital/v2` y
atributo raíz `version="2.0"`. v2 **coexiste** con v1: los XSD de v1 quedan
intactos en `xsd/` y los de v2 viven en `xsd/v2/`.

### Agregado
- Contenedor `<Actos>` con `<Acto numero="N">` (`minOccurs=1`, sin tope). Actos
  de tipos heterogéneos permitidos. El atributo `numero` es entero positivo
  obligatorio (su unicidad la valida el servicio, no el XSD).
- `xsd/v2/comunes/parte.xsd`: tipo `ParteType` (= `PersonaType` + atributo `rol`)
  y enum `RolParteEnum` con `ADQUIRENTE`, `TRANSMITENTE`, `ACREEDOR`, `DEUDOR`
  (`ACREEDOR`/`DEUDOR` quedan listos para Hipoteca, aún no disponible).
- Ejemplo `ejemplos/v2/compraventa-dos-actos.xml`: testimonio con dos actos.
- Migración de los 6 ejemplos a `ejemplos/v2/`.
- Códigos de error de negocio: `NUMERO_ACTO_DUPLICADO`, `ROL_NO_VALIDO_PARA_ACTO`,
  `ACTO_SIN_ADQUIRENTE`.

### Cambiado
- **Bajan del nivel testimonio al nivel `<Acto>`**: `Partes` (antes
  `Adquirentes`/`Transmitentes`), `Inmuebles`, `DatosEconomicos`,
  `CertificacionCatastral`, `NomenclaturaCatastral` (opcional),
  `CertificacionRegistralPrevia` y `VisadoRentas`. Cada acto tiene su propio
  juego de estos bloques.
- **Partes con rol genérico**: se reemplazan los contenedores `Adquirentes` /
  `Transmitentes` de v1 por una lista de `<Parte rol="...">`. `CompraventaType`
  ya no contiene partes ni inmuebles: solo los campos propios del tipo
  (`DescripcionActoIncompleto` y los textos libres).
- `PROPORCIONES_NO_SUMAN_UNO`: la suma de proporciones de adquirentes se valida
  **por acto**.
- Documentación de `docs/` reescrita a v2.

### Quedan a nivel testimonio (sin cambios respecto de v1)
- `MetadatosEnvio`, `EscribanoAutorizante`, `Otorgamiento`, `Rogante`,
  `TextoCuerpo`, `Observaciones` y `<ds:Signature>` (un trámite, un cuerpo de
  escritura por testimonio).

---

## [No publicado]

### Cambios mayores en el contrato (alineamiento al Excel del RPI)

#### Agregado
- Elemento `<TextoCuerpo>` obligatorio: transcripción del cuerpo del acto
  en texto plano (max 500.000 caracteres). Hijo directo de
  `<TestimonioDigital>`, ubicado después de `<Rogante>` y antes de
  `<Observaciones>`. Usado por el servicio del RPI para análisis NLP,
  visualización y búsqueda full-text.
- Soporte para personas jurídicas (Tipo=J) y organismos públicos (Tipo=O).
- Bloque `Representante` opcional en personas (adquirentes y transmitentes).
- Bloque `CertificacionCatastral` con campos condicionales.
- Bloque `NomenclaturaCatastral` opcional con 5 subcampos.
- Bloque `VisadoRentas` obligatorio.
- Campos en `<Compraventa>`: `DescripcionActoIncompleto`, `ReconocimientoHipotecaMedidasCautelares`, `AfectacionesAlDominio`, `AsentimientoConyugal`.
- Campos de contacto obligatorios en `Rogante`: `NumeroRegistro`, `Localidad`, `Provincia`, `Domicilio`, `Telefono`.
- Campos opcionales en `Persona` (humana): `Nupcias`, `Conyuge`, `InscripcionOrganismoSede`.
- En `IdentificacionInmueble`: campos `Barra`, `Tomo`, `Folio`, `Finca`.
- Tres ejemplos nuevos: `compraventa-persona-juridica.xml`, `compraventa-con-representante.xml`, `compraventa-inmueble-antiguo.xml`.
- Documento `docs/10-campos-del-formulario.md` con tabla plana de campos.

#### Cambiado
- `xsd/xmldsig-core-schema.xsd`: cambio de `namespace="##any"` a
  `namespace="##other"` en `SignatureMethodType`, `CanonicalizationMethodType`
  y `DigestMethodType`. Corrige violación de Unique Particle Attribution
  (UPA) que hace que el XSD canónico de W3C falle con validadores
  estrictos como xmllint.
- `xsd/testimonio-digital.xsd`: agregado atributo opcional `Id` (xs:ID)
  al elemento raíz `TestimonioDigital`. Necesario para compatibilidad
  con librerías de firma XML-DSig que agregan automáticamente este
  atributo (xml-crypto, Apache Santuario, etc.).
- `docs/05-firma-digital.md`: agregada sección sobre compatibilidad con
  firmadores que agregan `Id`. Agregada nota sobre el estado actual de
  verificación de certificado en v1.0.
- `IdentificacionInmueble/Matricula`: ahora es `xs:integer` 1-8 dígitos (antes string "DD-NNNN").
- `IdentificacionInmueble/Departamento`: ahora es código numérico 1-16 con enum (antes string libre).
- `IdentificacionInmueble`: campo `Barrio` renombrado a `Barra` (alineamiento al Excel).
- `CertificacionRegistralPrevia/FechaEmision` renombrada a `FechaEmisionPrimera`.
- `CertificacionRegistralPrevia/FechaVigencia` renombrada a `FechaEmisionSegunda`.
- `EscribanoAutorizante/Nombre`: longitud max ajustada a 60 (antes 120).
- `EscribanoAutorizante/RegistroNumero`: longitud max ajustada a 8 (antes 10).
- `EscribanoAutorizante/Sede`: longitud max ajustada a 40 (antes 60).
- `Persona/Proporcion`: ahora string fracción "N/D" (antes atributos numerador/denominador).
- Enum `TipoDocumento`: removido `CI` (alineamiento al Excel: solo DNI, LE, LC, PAS).
- Archivo `xsd/comunes/persona-humana.xsd` renombrado a `xsd/comunes/persona.xsd`.

#### Removido
- `Persona/Documento` con atributo `tipo`: reemplazado por `TipoDocumento` y `NumeroDocumento` como elementos separados (alineamiento al Excel).

---

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
