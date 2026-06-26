# 04 — Formato XML

## Estructura general

El cambio central de v2 es que un testimonio contiene **N actos (1 a N)**. Los
bloques sustantivos (partes, inmuebles, datos económicos, certificaciones,
visado) **bajan del nivel testimonio al nivel de cada `<Acto>`**. A nivel
testimonio quedan los datos que son únicos por trámite (un escribano, un
otorgamiento, un cuerpo de escritura, una firma).

El XML del testimonio digital tiene esta estructura raíz:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<TestimonioDigital
    xmlns="https://contrato.rpi.jusneuquen.gov.ar/testimonio-digital/v2"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:ds="http://www.w3.org/2000/09/xmldsig#"
    xsi:schemaLocation="https://contrato.rpi.jusneuquen.gov.ar/testimonio-digital/v2 testimonio-digital.xsd"
    version="2.0">

  <!-- ── Nivel testimonio: un trámite, un cuerpo de escritura ── -->

  <MetadatosEnvio>
    <!-- Identificación del envío, hash del PDF, timestamp -->
  </MetadatosEnvio>

  <EscribanoAutorizante>
    <!-- Datos del escribano que firma el testimonio -->
  </EscribanoAutorizante>

  <Otorgamiento>
    <!-- Lugar, fecha, número de escritura, folio del protocolo -->
  </Otorgamiento>

  <!-- ── Lista de actos: 1..N, de tipos heterogéneos ── -->
  <Actos>
    <Acto numero="1">
      <!-- Discriminador del tipo de acto. v2.0: solo Compraventa -->
      <Compraventa>
        <!-- Bloques de texto libre opcionales del acto -->
      </Compraventa>
      <Partes>
        <!-- Una o varias <Parte rol="ADQUIRENTE|TRANSMITENTE|..."> -->
      </Partes>
      <Inmuebles>
        <Inmueble>
          <IdentificacionInmueble>
            <!-- Matrícula o terna Tomo/Folio/Finca + departamento -->
          </IdentificacionInmueble>
          <CertificacionCatastral>
            <!-- Catastral del inmueble (obligatoria por inmueble) -->
          </CertificacionCatastral>
          <NomenclaturaCatastral>
            <!-- Nomenclatura en 5 campos (obligatoria por inmueble) -->
          </NomenclaturaCatastral>
        </Inmueble>
        <!-- Más <Inmueble> si el acto los tiene; cada uno con su catastro -->
      </Inmuebles>
      <DatosEconomicos>
        <!-- Valuación fiscal, monto, moneda, cotización -->
      </DatosEconomicos>
      <CertificacionDominio>
        <!-- Certificado de dominio previo (sobre el inmueble): Numero + FechaEmision -->
      </CertificacionDominio>
      <CertificacionInhibicion>
        <!-- Certificado de inhibición previo (sobre la persona): Numero + FechaEmision -->
      </CertificacionInhibicion>
      <VisadoRentas>
        <!-- Visado de Rentas (obligatorio) -->
      </VisadoRentas>
    </Acto>
    <!-- <Acto numero="2"> ... </Acto>  (más actos si la escritura los tiene) -->
  </Actos>

  <!-- ── Vuelta al nivel testimonio ── -->

  <Rogante>
    <!-- Persona que rogará, con datos de contacto -->
  </Rogante>

  <TextoCuerpo>
    <!-- Transcripción del cuerpo completo de la escritura (obligatorio) -->
  </TextoCuerpo>

  <Observaciones>
    <!-- Opcional, texto libre -->
  </Observaciones>

  <ds:Signature>
    <!-- Firma XML-DSig del escribano autorizante -->
  </ds:Signature>

</TestimonioDigital>
```

> El orden de los elementos dentro de `<Acto>` es fijo: primero el discriminador
> del tipo (`Compraventa`), luego `Partes`, `Inmuebles`, `DatosEconomicos`,
> `CertificacionDominio`, `CertificacionInhibicion` y `VisadoRentas`. Dentro de
> cada `<Inmueble>` el orden también es fijo: `IdentificacionInmueble`,
> `CertificacionCatastral`, `NomenclaturaCatastral`.

> El atributo `numero` de cada `<Acto>` es un entero positivo obligatorio. La
> **unicidad** de `numero` entre los actos no la valida el XSD: la valida el
> servicio del RPI.

> Las validaciones condicionales entre campos (por ejemplo, "si moneda=USD
> entonces cotización obligatoria") **no** las hace el XSD: las hace el servicio
> del RPI. Ver [07 — Respuestas y errores](07-respuestas-y-errores.md).

## Namespace

El namespace del contrato es:

```
https://contrato.rpi.jusneuquen.gov.ar/testimonio-digital/v2
```

Todos los elementos del testimonio (excepto la firma XML-DSig, que usa su
propio namespace) están bajo este namespace.

La versión MAJOR del contrato está embebida en el namespace (`/v2`). El
namespace de v1 (`/v1`) y sus XSD siguen existiendo en paralelo en `xsd/`: un
XML de v1 sigue siendo válido contra el XSD de v1. Esta documentación describe
la v2.0.

## Secciones del XML

### MetadatosEnvio

Datos de control del envío:

- `IdentificadorEnvio`: UUID v4 generado por el sistema del Colegio. **Único**
  por testimonio. Si se reenvía, debe ser el mismo (para idempotencia).
- `TimestampEnvio`: fecha y hora del envío en formato ISO 8601 con timezone.
- `HashPDF`: hash SHA-256 del PDF que viaja en la misma petición, en
  hexadecimal lowercase.
- `VersionContrato`: la versión del contrato XSD que el cliente declara usar.

```xml
<MetadatosEnvio>
  <IdentificadorEnvio>550e8400-e29b-41d4-a716-446655440000</IdentificadorEnvio>
  <TimestampEnvio>2026-06-15T10:23:45-03:00</TimestampEnvio>
  <HashPDF>3a7bd3e2360a3d29eea436fcfb7e44c735d117c42d1c1835420b6b9942dd4f1b</HashPDF>
  <VersionContrato>2.0</VersionContrato>
</MetadatosEnvio>
```

### EscribanoAutorizante

Datos del escribano que autoriza la escritura. Es la persona cuya firma
digital firma el XML.

```xml
<EscribanoAutorizante>
  <Nombre>JUAN CARLOS PÉREZ</Nombre>
  <CUIT>20-12345678-9</CUIT>
  <RegistroNumero>123</RegistroNumero>
  <Sede>NEUQUÉN</Sede>
</EscribanoAutorizante>
```

### Otorgamiento

Datos del acto notarial en sí:

```xml
<Otorgamiento>
  <Lugar>NEUQUÉN</Lugar>
  <Fecha>2026-06-10</Fecha>
  <NumeroEscritura>00012345</NumeroEscritura>
  <FolioProtocolo>00000123</FolioProtocolo>
</Otorgamiento>
```

### Actos / Acto

`Actos` es el contenedor de los **N actos** del testimonio. Tiene uno o más
elementos `<Acto numero="...">`. Los actos pueden ser de tipos distintos entre
sí (en v2.0 el único tipo disponible es `Compraventa`).

Cada `<Acto>` contiene, en este orden:

1. El **discriminador del tipo de acto** (`Compraventa`).
2. `Partes` — las personas que intervienen, con su rol por atributo.
3. `Inmuebles` — uno o más `<Inmueble>`, cada uno con su `IdentificacionInmueble`,
   su `CertificacionCatastral` y su `NomenclaturaCatastral` (el catastro es **por
   inmueble**).
4. `DatosEconomicos`, `CertificacionDominio`, `CertificacionInhibicion` y
   `VisadoRentas`.

Estos bloques en v1 vivían a nivel testimonio; en v2 **bajan al acto**, porque
cada acto puede tener distintas partes, inmuebles, monto y certificaciones.

```xml
<Actos>
  <Acto numero="1">
    <Compraventa>
      <DescripcionActoIncompleto>Compraventa con saldo de precio</DescripcionActoIncompleto>
    </Compraventa>
    <Partes> ... </Partes>
    <Inmuebles>
      <Inmueble>
        <IdentificacionInmueble> ... </IdentificacionInmueble>
        <CertificacionCatastral> ... </CertificacionCatastral>
        <NomenclaturaCatastral> ... </NomenclaturaCatastral>
      </Inmueble>
    </Inmuebles>
    <DatosEconomicos> ... </DatosEconomicos>
    <CertificacionDominio> ... </CertificacionDominio>
    <CertificacionInhibicion> ... </CertificacionInhibicion>
    <VisadoRentas> ... </VisadoRentas>
  </Acto>
</Actos>
```

El catastro (`CertificacionCatastral` y `NomenclaturaCatastral`) vive **dentro de
cada `<Inmueble>`**; el resto de los bloques (`CertificacionDominio`,
`CertificacionInhibicion`, `DatosEconomicos`, `VisadoRentas`, `Compraventa`,
`Partes` e `Inmuebles`) viven **dentro de cada `<Acto>`**.

### CertificacionDominio y CertificacionInhibicion

Las certificaciones registrales previas que el escribano solicitó al RPI antes
de otorgar la escritura. Son **dos certificados distintos**, modelados como dos
bloques separados a nivel acto:

- **`CertificacionDominio`**: recae sobre el **inmueble** (estado de dominio,
  gravámenes, restricciones).
- **`CertificacionInhibicion`**: recae sobre la **persona** (si el transmitente
  está inhibido para disponer de sus bienes).

Cada uno tiene la misma estructura simple: `Numero` + `FechaEmision`. Reemplazan
al antiguo `CertificacionRegistralPrevia` (un número con dos fechas heredado del
legacy). Estructuralmente son opcionales en el XSD, pero **para compraventa el
servicio del RPI exige ambos**.

```xml
<CertificacionDominio>
  <Numero>2026-001234</Numero>
  <FechaEmision>2026-06-01</FechaEmision>
</CertificacionDominio>
<CertificacionInhibicion>
  <Numero>INH-2026-000789</Numero>
  <FechaEmision>2026-06-02</FechaEmision>
</CertificacionInhibicion>
```

### CertificacionCatastral (por inmueble)

Certificación catastral del inmueble (distinta de la registral). Vive **dentro
de cada `<Inmueble>`** y es **obligatoria por inmueble**: si un acto tiene varios
inmuebles, cada uno lleva la suya. Si `Emitido` es `false`, los demás campos
pueden omitirse. Si es `true`, `Numero` y `CodigoValidacion` son obligatorios (lo
valida el servicio). `Observaciones` es obligatorio solo si
`TieneObservaciones=true`.

```xml
<CertificacionCatastral>
  <Emitido>true</Emitido>
  <Numero>CAT-2026-5678</Numero>
  <CodigoValidacion>VAL-ABCD-1234</CodigoValidacion>
  <TieneObservaciones>false</TieneObservaciones>
</CertificacionCatastral>
```

Caso sin certificación emitida:

```xml
<CertificacionCatastral>
  <Emitido>false</Emitido>
</CertificacionCatastral>
```

### NomenclaturaCatastral (por inmueble)

Identificación parcelaria del inmueble. Vive **dentro de cada `<Inmueble>`** y es
**obligatoria por inmueble**. Los 5 campos son obligatorios, con longitudes fijas
(2, 2, 3, 4 y 4 caracteres).

```xml
<NomenclaturaCatastral>
  <Campo1>09</Campo1>
  <Campo2>21</Campo2>
  <Campo3>045</Campo3>
  <Campo4>0123</Campo4>
  <Campo5>0008</Campo5>
</NomenclaturaCatastral>
```

### DatosEconomicos

Valuación fiscal del inmueble y monto del acto:

```xml
<DatosEconomicos>
  <ValuacionFiscal>
    <valor>5000000.00</valor>
  </ValuacionFiscal>
  <Monto>
    <valor>50000000.00</valor>
    <moneda>$</moneda>
  </Monto>
</DatosEconomicos>
```

Para compraventas en moneda extranjera, incluir la cotización al día del acto:

```xml
<Monto>
  <valor>50000.00</valor>
  <moneda>USD</moneda>
  <cotizacion>1000.50</cotizacion>
</Monto>
```

### VisadoRentas

Visado de la Dirección Provincial de Rentas. Bloque **obligatorio**.

- `Tipo=R`: con visado. `NumeroTramite` es obligatorio (lo valida el servicio).
- `Tipo=A`: sin visado (exento u otra causal). `NumeroTramite` no aplica.

```xml
<VisadoRentas>
  <Tipo>R</Tipo>
  <NumeroTramite>RNT-987654</NumeroTramite>
</VisadoRentas>
```

### Compraventa (discriminador del acto)

Primer elemento de cada `<Acto>`. Identifica el tipo de acto (en v2.0, único
tipo) y lleva solo los datos **propios del tipo compraventa**. Las partes y los
inmuebles ya **no** viven dentro de `Compraventa`: viven en `Acto/Partes` y
`Acto/Inmuebles`. Subelementos de `Compraventa` (todos opcionales):

- `DescripcionActoIncompleto`: aclaración del escribano si no es una compraventa
  estándar completa.
- `ReconocimientoHipotecaMedidasCautelares` (texto libre).
- `AfectacionesAlDominio` (texto libre).
- `AsentimientoConyugal` (texto libre).

Una compraventa estándar y completa puede llevar `<Compraventa/>` vacío.

Ver `xsd/v2/actos/compraventa.xsd` para el detalle.

### Partes

Lista de personas que intervienen en el acto. Reemplaza los contenedores
`Adquirentes` / `Transmitentes` de v1 por una lista única de `<Parte>`, donde el
rol se expresa con el atributo obligatorio `rol`. Una o más `Parte`.

```xml
<Partes>
  <Parte rol="TRANSMITENTE">
    <!-- contenido de Persona (Tipo, ApellidoODenominacion, CUIT, ...) -->
  </Parte>
  <Parte rol="ADQUIRENTE">
    <!-- contenido de Persona, con Proporcion -->
  </Parte>
</Partes>
```

Roles definidos: `ADQUIRENTE`, `TRANSMITENTE`, `ACREEDOR`, `DEUDOR`
(`ACREEDOR`/`DEUDOR` quedan listos para Hipoteca, aún no disponible). El XSD
acepta **cualquier** rol en cualquier acto; que el rol corresponda al tipo de
acto lo valida el servicio del RPI.

### Inmuebles

Uno o más `<Inmueble>` (hasta 50 por acto). Cada `<Inmueble>` contiene, en este
orden: su `IdentificacionInmueble`, su `CertificacionCatastral` y su
`NomenclaturaCatastral`. El catastro es **por inmueble** (obligatorio en cada
uno): si el acto tiene varios inmuebles, cada uno lleva su propio catastro y su
nomenclatura. Ver [Identificación del inmueble](#identificación-del-inmueble).

### Rogante

Persona que actúa como rogante del trámite ante el RPI. Típicamente es el
escribano autorizante. Los datos de contacto son **obligatorios**.

```xml
<Rogante>
  <CUIT>20-12345678-9</CUIT>
  <Nombre>JUAN CARLOS PÉREZ</Nombre>
  <NumeroRegistro>123</NumeroRegistro>
  <Localidad>NEUQUÉN</Localidad>
  <Provincia>NEUQUÉN</Provincia>
  <Domicilio>CALLE FALSA 123</Domicilio>
  <Telefono>0299-4567890</Telefono>
</Rogante>
```

### TextoCuerpo

Transcripción del cuerpo del acto en texto plano. Es **obligatorio**.

El `TextoCuerpo` lleva el texto completo del cuerpo de la escritura: la
comparecencia de las partes, las cláusulas del acto, los asentimientos
que figuren en el cuerpo, y el cierre del escribano. Es la versión
textual estructurada de lo que aparece en el PDF firmado adjunto.

**Formato esperado**: texto plano. No se espera HTML, Markdown, XML
embebido ni otros marcados. Los saltos de línea del texto natural se
preservan tal cual.

**Longitud**: hasta 500.000 caracteres. Cubre testimonios de hasta
aproximadamente 150 páginas de texto plano.

**Responsabilidad**: la fidelidad del `TextoCuerpo` respecto al cuerpo de
la escritura del PDF es responsabilidad del escribano autorizante. El RPI
**no valida** la coincidencia textual entre XML y PDF. La integridad
legal del testimonio está dada por las firmas digitales del XML y del
PDF, no por la coincidencia textual entre ambos.

**Uso del servicio del RPI**: el RPI procesa el `TextoCuerpo` para
análisis NLP, visualización en interfaces internas, y búsqueda
full-text. Es el insumo principal para estos casos de uso.

Ejemplo:

```xml
<TextoCuerpo>En la ciudad de Neuquén, capital de la Provincia del Neuquén,
República Argentina, a los diez días del mes de junio del año dos mil
veintiséis, ante mí, JUAN CARLOS PÉREZ, escribano titular del Registro
Notarial Número 123...

[continúa el cuerpo completo de la escritura]</TextoCuerpo>
```

### Observaciones (opcional)

Texto libre que el escribano quiere transmitir al RPI (hasta 4000 caracteres).
Es distinto de `AsentimientoConyugal`, `ReconocimientoHipotecaMedidasCautelares`
y `AfectacionesAlDominio`, que viven dentro de `<Compraventa>`.

```xml
<Observaciones>
  Se aclara que el inmueble es de origen ganancial...
</Observaciones>
```

### Signature

Firma XML-DSig del escribano. Detalle en [05 — Firma digital](05-firma-digital.md).

## La Parte (Persona + rol)

Cada `<Parte>` **es** una persona (el tipo `Persona`, reutilizado sin cambios de
v1) más el atributo obligatorio `rol`. El contenido de `<Parte>` es exactamente
el de una `Persona` (los mismos subelementos), con `rol` en el elemento de
apertura. Por eso los campos descritos abajo aplican igual a cualquier parte,
sea cual sea su rol.

`Persona` discrimina por `Tipo`:

| `Tipo` | Significado | Notas |
|--------|-------------|-------|
| `H` | Persona humana | Requiere `Nombres` y `TipoDocumento`. |
| `J` | Persona jurídica | `ApellidoODenominacion` es la razón social. No usa `Nombres`. `NumeroDocumento` repite el CUIT sin guiones. |
| `O` | Organismo público | Igual que `J`. |

Campos comunes (todos los tipos): `Tipo`, `ApellidoODenominacion`, `CUIT`,
`NumeroDocumento`, `PEP`.

Campos solo para humanas (`Tipo=H`): `Nombres`, `TipoDocumento`, `EstadoCivil`,
`Nupcias`, `Conyuge`, `Nacionalidad`, `FechaNacimiento`. Para una parte con rol
**ADQUIRENTE** humana, `EstadoCivil`, `Nacionalidad` y `FechaNacimiento` son
obligatorios; para una **TRANSMITENTE** humana son opcionales (lo valida el
servicio).

Campos solo para `J`/`O`: `InscripcionOrganismoSede`.

`Proporcion` (string fracción `"N/D"`, ej. `"1/2"`) aplica a las partes que
adquieren (rol `ADQUIRENTE`). Las reglas "qué roles admiten proporción" y "las
proporciones suman 1 **por acto**" las valida el servicio, no el XSD.

Ejemplo de parte humana adquirente:

```xml
<Parte rol="ADQUIRENTE">
  <Tipo>H</Tipo>
  <ApellidoODenominacion>GONZÁLEZ</ApellidoODenominacion>
  <Nombres>MARÍA LAURA</Nombres>
  <CUIT>27-28456789-3</CUIT>
  <TipoDocumento>DNI</TipoDocumento>
  <NumeroDocumento>28456789</NumeroDocumento>
  <EstadoCivil>SOLTERA</EstadoCivil>
  <Nacionalidad>ARGENTINA</Nacionalidad>
  <FechaNacimiento>1981-03-15</FechaNacimiento>
  <PEP>false</PEP>
  <Proporcion>1/1</Proporcion>
</Parte>
```

Ejemplo de parte jurídica transmitente (`NumeroDocumento` repite el CUIT sin
guiones; `TipoDocumento` se omite):

```xml
<Parte rol="TRANSMITENTE">
  <Tipo>J</Tipo>
  <ApellidoODenominacion>INMOBILIARIA DEL SUR S.A.</ApellidoODenominacion>
  <CUIT>30-70123456-7</CUIT>
  <NumeroDocumento>30701234567</NumeroDocumento>
  <PEP>false</PEP>
  <InscripcionOrganismoSede>IGJ NEUQUÉN</InscripcionOrganismoSede>
</Parte>
```

### Representante (bloque opcional)

Si la persona actúa por medio de un representante (tutor, apoderado, etc.), se
incluye el bloque `Representante`. Si no, se omite por completo. El tipo de
documento del representante admite solo `DNI`, `LE` o `LC` (no `PAS`).

```xml
<Representante>
  <Apellido>MORALES</Apellido>
  <Nombres>GUSTAVO ADRIÁN</Nombres>
  <CUIT>20-27890123-5</CUIT>
  <TipoDocumento>DNI</TipoDocumento>
  <NumeroDocumento>27890123</NumeroDocumento>
  <PEP>false</PEP>
</Representante>
```

## Identificación del inmueble

`IdentificacionInmueble` es el primer hijo de cada `<Inmueble>`; le siguen, en el
mismo `<Inmueble>`, la `CertificacionCatastral` y la `NomenclaturaCatastral` de
ese inmueble (ver secciones anteriores). `IdentificacionInmueble` soporta dos
modos. El servicio exige que se complete **al menos uno**:

- **Por matrícula**: `Matricula` (entero, hasta 8 dígitos) + `Departamento`.
- **Por tomo/folio/finca**: para inmuebles antiguos sin matrícula.

`Departamento` es siempre obligatorio y se expresa como código numérico (1 a
16). `Barra` (barra catastral) es opcional.

```xml
<IdentificacionInmueble>
  <Matricula>3456</Matricula>
  <Departamento>5</Departamento>
</IdentificacionInmueble>
```

Inmueble antiguo sin matrícula:

```xml
<IdentificacionInmueble>
  <Departamento>16</Departamento>
  <Tomo>145</Tomo>
  <Folio>231</Folio>
  <Finca>8902</Finca>
</IdentificacionInmueble>
```

Un `<Inmueble>` completo incluye, además de la identificación, su catastro
(certificación + nomenclatura), que son **obligatorios por inmueble**:

```xml
<Inmueble>
  <IdentificacionInmueble>
    <Matricula>3456</Matricula>
    <Departamento>5</Departamento>
  </IdentificacionInmueble>
  <CertificacionCatastral>
    <Emitido>true</Emitido>
    <Numero>CAT-2026-5678</Numero>
    <CodigoValidacion>VAL-ABCD-1234</CodigoValidacion>
    <TieneObservaciones>false</TieneObservaciones>
  </CertificacionCatastral>
  <NomenclaturaCatastral>
    <Campo1>05</Campo1>
    <Campo2>20</Campo2>
    <Campo3>001</Campo3>
    <Campo4>0100</Campo4>
    <Campo5>0001</Campo5>
  </NomenclaturaCatastral>
</Inmueble>
```

### Códigos de departamento

| Código | Nombre | Código | Nombre |
|--------|--------|--------|--------|
| 1 | Aluminé | 9 | Loncopué |
| 2 | Añelo | 10 | Los Lagos |
| 3 | Catán Lil | 11 | Minas |
| 4 | Collón Curá | 12 | Ñorquín |
| 5 | Confluencia | 13 | Pehuenches |
| 6 | Chos Malal | 14 | Picunches |
| 7 | Huiliches | 15 | Picún Leufú |
| 8 | Lácar | 16 | Zapala |

## Tipos comunes

Algunos tipos se reutilizan entre actos. Por eso están en archivos XSD
separados bajo `xsd/v2/comunes/`:

- **Parte** (`parte.xsd`): `Persona` + atributo `rol`. Define el enum de roles.
- **Persona**: contenido de cada `Parte` (reutilizado tal cual de v1).
- **IdentificacionInmueble**, **CertificacionCatastral**, **NomenclaturaCatastral**:
  bloques comunes del inmueble (los tres viven dentro de cada `<Inmueble>`).
- **CertificacionDominio**, **CertificacionInhibicion**, **VisadoRentas**,
  **DatosEconomicos**: bloques comunes del acto.

Cuando agreguemos nuevos actos (hipoteca, donación), van a reutilizar estos
mismos tipos sin redefinirlos: el acto nuevo solo define sus campos propios, ya
que partes e inmuebles viven en `Acto`.

## Validación

El XML debe validar contra `xsd/v2/testimonio-digital.xsd`. La validación
incluye:

- Estructura (todos los elementos requeridos presentes).
- Tipos de datos (fechas válidas, montos numéricos, `numero` entero positivo, etc.).
- Cardinalidades (al menos un acto, al menos una parte por acto, al menos un
  inmueble por acto, una `CertificacionCatastral` y una `NomenclaturaCatastral`
  por inmueble, etc.).
- Restricciones de longitud y formato.

Las validaciones cruzadas las hace el servicio del RPI **después** del XSD:
unicidad de `numero` entre actos, que el rol corresponda al tipo de acto,
proporciones que suman 1 por acto, condicionales por `Tipo` de persona,
matrícula vs. terna, etc. Si el XML no valida contra el XSD, el RPI responde
HTTP 400; si falla una validación de negocio, responde HTTP 422 (ver
[07 — Respuestas y errores](07-respuestas-y-errores.md)).

## Encoding

El XML debe estar en **UTF-8** con BOM opcional. El RPI acepta ambos
(con y sin BOM).

## Próximos pasos

- Para la lista plana de todos los campos del formulario, andá a
  [10 — Campos del formulario](10-campos-del-formulario.md).
- Para la firma digital, andá a [05 — Firma digital](05-firma-digital.md).
- Para el manejo del PDF, andá a [06 — Adjunto PDF](06-adjunto-pdf.md).
- Para ver XMLs reales, mirá los archivos en `../ejemplos/`.
