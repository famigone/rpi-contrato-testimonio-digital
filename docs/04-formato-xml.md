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

  <DatosEconomicos>
    <!-- Valuación fiscal, monto, moneda, cotización -->
  </DatosEconomicos>

  <Acto>
    <!-- Datos específicos del tipo de acto. v1.0: solo Compraventa -->
    <Compraventa>
      <Adquirentes>
        <!-- Una o varias PersonaHumana con proporción -->
      </Adquirentes>
      <Transmitentes>
        <!-- Uno o varios PersonaHumana -->
      </Transmitentes>
      <Inmuebles>
        <!-- Uno o varios IdentificacionInmueble -->
      </Inmuebles>
    </Compraventa>
  </Acto>

  <Rogante>
    <!-- Persona que rogará (típicamente el escribano autorizante) -->
  </Rogante>

  <Observaciones>
    <!-- Opcional, texto libre -->
  </Observaciones>

  <ds:Signature>
    <!-- Firma XML-DSig del escribano autorizante -->
  </ds:Signature>

</TestimonioDigital>
```

## Namespace

El namespace del contrato es:

```
https://contrato.rpi.jusneuquen.gov.ar/testimonio-digital/v1
```

Todos los elementos del testimonio (excepto la firma XML-DSig, que usa su
propio namespace) están bajo este namespace.

La versión del contrato está embebida en el namespace (`/v1`). Cuando salga
v2 con nuevos tipos de actos, el namespace cambiará a `/v2` y los XML viejos
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

Ejemplo:

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
la escritura:

```xml
<CertificacionRegistralPrevia>
  <Numero>2026-001234</Numero>
  <FechaEmision>2026-06-01</FechaEmision>
  <FechaVigencia>2026-06-30</FechaVigencia>
</CertificacionRegistralPrevia>
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

### Acto / Compraventa

Datos específicos de la compraventa (en v1.0, único tipo de acto). Tres
subelementos obligatorios:

- `Adquirentes`: uno o más, con proporción de adquisición.
- `Transmitentes`: uno o más.
- `Inmuebles`: uno o más.

Ver `xsd/actos/compraventa.xsd` para el detalle.

### Rogante

Persona que actúa como rogante del trámite ante el RPI. Típicamente es el
escribano autorizante, pero podría ser otra persona designada.

```xml
<Rogante>
  <CUIT>20-12345678-9</CUIT>
  <Nombre>JUAN CARLOS PÉREZ</Nombre>
</Rogante>
```

### Observaciones (opcional)

Texto libre que el escribano quiere transmitir al RPI:

```xml
<Observaciones>
  Se aclara que el inmueble es de origen ganancial...
</Observaciones>
```

### Signature

Firma XML-DSig del escribano. Detalle en [05 — Firma digital](05-firma-digital.md).

## Tipos comunes

Algunos tipos se reutilizan entre actos. Por eso están en archivos XSD
separados bajo `xsd/comunes/`:

- **PersonaHumana**: usada para adquirentes, transmitentes, rogante.
- **IdentificacionInmueble**: matrícula, departamento, ubicación.

Cuando agreguemos nuevos actos (hipoteca, donación), van a reutilizar estos
mismos tipos sin redefinirlos.

## Validación

El XML debe validar contra `xsd/testimonio-digital.xsd`. La validación
incluye:

- Estructura (todos los elementos requeridos presentes).
- Tipos de datos (fechas válidas, montos numéricos, etc.).
- Cardinalidades (al menos un adquirente, al menos un inmueble, etc.).
- Restricciones de longitud y formato.

Si el XML no valida, el RPI responde con HTTP 400 e indica qué falló
(ver [07 — Respuestas y errores](07-respuestas-y-errores.md)).

## Encoding

El XML debe estar en **UTF-8** con BOM opcional. El RPI acepta ambos
(con y sin BOM).

## Próximos pasos

- Para la firma digital, andá a [05 — Firma digital](05-firma-digital.md).
- Para el manejo del PDF, andá a [06 — Adjunto PDF](06-adjunto-pdf.md).
- Para ver XMLs reales, mirá los archivos en `../ejemplos/`.
