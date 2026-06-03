# 04 — Formato XML

## Estructura general

El XML del testimonio digital tiene esta estructura raíz:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<TestimonioDigital
    xmlns="https://contrato.rpi.jusneuquen.gov.ar/testimonio-digital/v1"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:ds="http://www.w3.org/2000/09/xmldsig#"
    xsi:schemaLocation="https://contrato.rpi.jusneuquen.gov.ar/testimonio-digital/v1 testimonio-digital.xsd"
    version="1.0">

  <MetadatosEnvio>
    <!-- Identificación del envío, hash del PDF, timestamp -->
  </MetadatosEnvio>

  <EscribanoAutorizante>
    <!-- Datos del escribano que firma el acto -->
  </EscribanoAutorizante>

  <Otorgamiento>
    <!-- Lugar, fecha, número de escritura, folio del protocolo -->
  </Otorgamiento>

  <CertificacionRegistralPrevia>
    <!-- Certificación de dominio previa al acto -->
  </CertificacionRegistralPrevia>

  <CertificacionCatastral>
    <!-- Certificación catastral del inmueble (obligatoria) -->
  </CertificacionCatastral>

  <NomenclaturaCatastral>
    <!-- Nomenclatura catastral en 5 campos (bloque opcional) -->
  </NomenclaturaCatastral>

  <DatosEconomicos>
    <!-- Valuación fiscal, monto, moneda, cotización -->
  </DatosEconomicos>

  <VisadoRentas>
    <!-- Visado de Rentas (obligatorio) -->
  </VisadoRentas>

  <Acto>
    <!-- Datos específicos del tipo de acto. v1.0: solo Compraventa -->
    <Compraventa>
      <Adquirentes>
        <!-- Una o varias Persona con proporción -->
      </Adquirentes>
      <Transmitentes>
        <!-- Una o varias Persona -->
      </Transmitentes>
      <Inmuebles>
        <!-- Uno o varios IdentificacionInmueble -->
      </Inmuebles>
      <!-- Bloques de texto libre opcionales -->
    </Compraventa>
  </Acto>

  <Rogante>
    <!-- Persona que rogará, con datos de contacto -->
  </Rogante>

  <Observaciones>
    <!-- Opcional, texto libre -->
  </Observaciones>

  <ds:Signature>
    <!-- Firma XML-DSig del escribano autorizante -->
  </ds:Signature>

</TestimonioDigital>
```

> Las validaciones condicionales entre campos (por ejemplo, "si moneda=USD
> entonces cotización obligatoria") **no** las hace el XSD: las hace el servicio
> del RPI. Ver [07 — Respuestas y errores](07-respuestas-y-errores.md).

## Namespace

El namespace del contrato es:

```
https://contrato.rpi.jusneuquen.gov.ar/testimonio-digital/v1
```

Todos los elementos del testimonio (excepto la firma XML-DSig, que usa su
propio namespace) están bajo este namespace.

La versión del contrato está embebida en el namespace (`/v1`). Cuando salga
v2 con cambios incompatibles, el namespace cambiará a `/v2` y los XML viejos
seguirán siendo válidos contra el XSD de v1.

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
  <VersionContrato>1.0</VersionContrato>
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

### CertificacionRegistralPrevia

La certificación de dominio que el escribano solicitó al RPI antes de otorgar
la escritura. Las dos fechas se llaman `FechaEmisionPrimera` y
`FechaEmisionSegunda` (alineadas al Excel del RPI).

```xml
<CertificacionRegistralPrevia>
  <Numero>2026-001234</Numero>
  <FechaEmisionPrimera>2026-06-01</FechaEmisionPrimera>
  <FechaEmisionSegunda>2026-06-30</FechaEmisionSegunda>
</CertificacionRegistralPrevia>
```

### CertificacionCatastral

Certificación catastral del inmueble (distinta de la registral). Bloque
**obligatorio**. Si `Emitido` es `false`, los demás campos pueden omitirse. Si
es `true`, `Numero` y `CodigoValidacion` son obligatorios (lo valida el
servicio). `Observaciones` es obligatorio solo si `TieneObservaciones=true`.

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

### NomenclaturaCatastral (opcional)

Identificación parcelaria del inmueble. Bloque **opcional**, pero si se incluye
los 5 campos son obligatorios, con longitudes fijas (2, 2, 3, 4 y 4 caracteres).

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

### Acto / Compraventa

Datos específicos de la compraventa (en v1.0, único tipo de acto). Subelementos:

- `DescripcionActoIncompleto` (opcional): aclaración del escribano si no es una
  compraventa estándar completa.
- `Adquirentes`: una o más `Persona`, con proporción de adquisición.
- `Transmitentes`: una o más `Persona`.
- `Inmuebles`: uno o más `IdentificacionInmueble`.
- `ReconocimientoHipotecaMedidasCautelares` (opcional, texto libre).
- `AfectacionesAlDominio` (opcional, texto libre).
- `AsentimientoConyugal` (opcional, texto libre).

Ver `xsd/actos/compraventa.xsd` para el detalle.

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

## El tipo Persona

`Persona` se reutiliza para adquirentes y transmitentes. Discrimina por `Tipo`:

| `Tipo` | Significado | Notas |
|--------|-------------|-------|
| `H` | Persona humana | Requiere `Nombres` y `TipoDocumento`. |
| `J` | Persona jurídica | `ApellidoODenominacion` es la razón social. No usa `Nombres`. `NumeroDocumento` repite el CUIT sin guiones. |
| `O` | Organismo público | Igual que `J`. |

Campos comunes (todos los tipos): `Tipo`, `ApellidoODenominacion`, `CUIT`,
`NumeroDocumento`, `PEP`.

Campos solo para humanas (`Tipo=H`): `Nombres`, `TipoDocumento`, `EstadoCivil`,
`Nupcias`, `Conyuge`, `Nacionalidad`, `FechaNacimiento`. Para **adquirente**
humano, `EstadoCivil`, `Nacionalidad` y `FechaNacimiento` son obligatorios; para
**transmitente** humano son opcionales (lo valida el servicio).

Campos solo para `J`/`O`: `InscripcionOrganismoSede`.

Solo para adquirentes: `Proporcion` (string fracción `"N/D"`, ej. `"1/2"`).

Ejemplo de persona humana adquirente:

```xml
<Persona>
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
</Persona>
```

Ejemplo de persona jurídica adquirente (`NumeroDocumento` repite el CUIT sin
guiones; `TipoDocumento` se omite):

```xml
<Persona>
  <Tipo>J</Tipo>
  <ApellidoODenominacion>INMOBILIARIA DEL SUR S.A.</ApellidoODenominacion>
  <CUIT>30-71234567-8</CUIT>
  <NumeroDocumento>30712345678</NumeroDocumento>
  <PEP>false</PEP>
  <InscripcionOrganismoSede>IGJ - Inspección General de Justicia, CABA</InscripcionOrganismoSede>
  <Proporcion>1/1</Proporcion>
</Persona>
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

`IdentificacionInmueble` soporta dos modos. El servicio exige que se complete
**al menos uno**:

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
separados bajo `xsd/comunes/`:

- **Persona**: usada para adquirentes y transmitentes.
- **IdentificacionInmueble**: matrícula/terna, departamento, barra.
- **CertificacionCatastral**, **NomenclaturaCatastral**, **VisadoRentas**:
  bloques comunes del testimonio.

Cuando agreguemos nuevos actos (hipoteca, donación), van a reutilizar estos
mismos tipos sin redefinirlos.

## Validación

El XML debe validar contra `xsd/testimonio-digital.xsd`. La validación
incluye:

- Estructura (todos los elementos requeridos presentes).
- Tipos de datos (fechas válidas, montos numéricos, etc.).
- Cardinalidades (al menos un adquirente, al menos un inmueble, etc.).
- Restricciones de longitud y formato.

Las validaciones cruzadas (proporciones que suman 1, condicionales por `Tipo`,
matrícula vs. terna, etc.) las hace el servicio del RPI **después** del XSD. Si
el XML no valida contra el XSD, el RPI responde HTTP 400; si falla una
validación de negocio, responde HTTP 422 (ver
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
