# XSD v2 — Testimonio Digital con N actos

v2 del contrato. Coexiste con v1 (que queda intacto en `xsd/`); v2 vive en
este `xsd/v2/` con su propio namespace.

## Namespace

```
https://contrato.rpi.jusneuquen.gov.ar/testimonio-digital/v2
```

Mismo host/esquema que v1, sufijo `/v2`. Consistente en todos los XSD de v2.
Todos los componentes se enlazan con `xs:include` (mismo namespace); el cliente
valida contra `xsd/v2/testimonio-digital.xsd` y los includes se resuelven solos.

## Qué cambia respecto de v1

Un testimonio pasa de **un acto** a **N actos (1..N)**:

- Aparece `<Actos>` con `<Acto numero="N">` (`minOccurs=1`, sin tope). Actos de
  tipos heterogéneos permitidos (el `xs:choice` por acto lo habilita).
- **Bajan del testimonio al acto**: `DatosEconomicos`, las certificaciones
  registrales y `VisadoRentas`, además de `Partes` e `Inmuebles`.
- **Quedan a nivel testimonio** (un trámite / un cuerpo de escritura, igual que
  v1): `MetadatosEnvio`, `EscribanoAutorizante`, `Otorgamiento`, `Rogante`,
  `TextoCuerpo`, `Observaciones`, `Signature`.
- **Partes con rol genérico**: se reemplazan los contenedores `Adquirentes` /
  `Transmitentes` de v1 por una lista de `<Parte rol="...">`. Ver `comunes/parte.xsd`.
- **Catastro por inmueble**: `CertificacionCatastral` y `NomenclaturaCatastral`
  ya no están a nivel acto: viven dentro de cada `<Inmueble>` (junto a
  `IdentificacionInmueble`) y son **obligatorias por inmueble**. Un acto con M
  inmuebles tiene M certificaciones catastrales y M nomenclaturas.
- **Certificación registral en dos bloques, según sobre qué recae cada uno**: el
  viejo `CertificacionRegistralPrevia` (número + dos fechas, heredado del legacy)
  se reemplaza por dos certificados, cada uno con `Numero` + `FechaEmision`:
  `CertificacionDominio` (sobre el **inmueble**) vive a **nivel acto**, y
  `CertificacionInhibicion` (sobre la **persona**) vive **dentro de cada
  `<Parte>` transmitente**. Opcionales en el XSD; obligatorios por regla de
  servicio (dominio para compraventa; inhibición por cada transmitente). Ver
  `comunes/certificacion-dominio.xsd` y `comunes/certificacion-inhibicion.xsd`.

## Estructura

```
TestimonioDigital (version="2.0")
├── MetadatosEnvio
├── EscribanoAutorizante
├── Otorgamiento
├── Actos
│   └── Acto (numero, 1..N)
│       ├── (choice)  Compraventa | …futuros…
│       ├── Partes
│       │   └── Parte (rol, 1..N)
│       │       ├── …campos de PersonaType (Tipo … PEP … Proporcion, Representante)
│       │       └── CertificacionInhibicion      (solo transmitentes; obligatoria por transmitente)
│       ├── Inmuebles
│       │   └── Inmueble (1..M)
│       │       ├── IdentificacionInmueble
│       │       ├── CertificacionCatastral      (obligatoria por inmueble)
│       │       └── NomenclaturaCatastral        (obligatoria por inmueble)
│       ├── DatosEconomicos
│       ├── CertificacionDominio                 (opcional XSD; obligatoria compraventa)
│       └── VisadoRentas
├── Rogante
├── TextoCuerpo
├── Observaciones (opcional)
└── ds:Signature
```

```
xsd/v2/
├── testimonio-digital.xsd          ← entry point (define ActoType, Actos, Inmueble, raíz)
├── comunes/
│   ├── parte.xsd                   ← ParteType (= PersonaType + rol + CertificacionInhibicion de transmitente); incluye certificacion-inhibicion.xsd
│   ├── persona.xsd                 ← reutilizado de v1 (PersonaType con Proporcion/Representante)
│   ├── identificacion-inmueble.xsd ← IdentificacionInmuebleType (dentro de Inmueble)
│   ├── certificacion-catastral.xsd ← CertificacionCatastralType (se referencia DENTRO de Inmueble)
│   ├── nomenclatura-catastral.xsd  ← NomenclaturaCatastralType (se referencia DENTRO de Inmueble)
│   ├── certificacion-dominio.xsd   ← CertificacionDominioType (Numero + FechaEmision; a nivel acto)
│   ├── certificacion-inhibicion.xsd← CertificacionInhibicionType (Numero + FechaEmision; DENTRO de cada <Parte> transmitente)
│   └── …                           ← resto de comunes (datos-economicos, visado-rentas, …)
└── actos/
    └── compraventa.xsd             ← CompraventaType SIN partes/inmuebles (viven en Acto)
```

`xmldsig-core-schema.xsd` se reutiliza por `xs:import` desde la copia compartida
en `../xmldsig-core-schema.xsd` (estándar W3C, dominio público).

## Modelo de partes (rol genérico)

`ParteType` **extiende** `PersonaType` y agrega el atributo obligatorio `rol`
(`RolParteEnum`) más el elemento opcional `CertificacionInhibicion` (último hijo
de la parte). No se rehace la persona: una `<Parte>` tiene exactamente el
contenido de `PersonaType` (incluida `Proporcion` y `Representante`), más `rol` y
—solo en los transmitentes— su `CertificacionInhibicion`. La inhibición es
opcional en el XSD; la regla "obligatoria por cada TRANSMITENTE, ausente en otros
roles" se valida en el servicio (coherente con ADR-002).

Roles definidos hoy: `ADQUIRENTE`, `TRANSMITENTE`, `ACREEDOR`, `DEUDOR`
(acreedor/deudor quedan listos aunque Hipoteca aún no esté en el choice).
Agregar un rol = una línea `<xs:enumeration>` en `parte.xsd`.

## Reglas que el XSD NO valida (van en código, coherente con ADR-002)

Intencional y documentado en los propios XSD:

- **Unicidad de `numero`** entre actos.
- **Qué roles corresponden a qué tipo de acto** (el XSD acepta cualquier
  `RolParte` en cualquier acto).
- **Qué roles admiten proporción** y que las **proporciones sumen 1 por acto**.
- **Que cada TRANSMITENTE lleve su `CertificacionInhibicion`** (el XSD la deja
  opcional en cualquier `<Parte>`; la obligatoriedad por transmitente la impone
  el servicio).
- Las validaciones condicionales heredadas de v1 (CUIT dígito verificador,
  fechas coherentes, campos condicionales por `Tipo` de persona, etc.).

## Cómo validar

```bash
xmllint --schema xsd/v2/testimonio-digital.xsd \
  ejemplos/v2/compraventa-dos-actos.xml --noout
```

`ejemplos/v2/compraventa-dos-actos.xml` es un ejemplo válido con dos actos
(personas humanas y jurídica con representante, monto en $ y en USD).

## Agregar un acto nuevo (p. ej. Hipoteca)

1. Crear `actos/hipoteca.xsd` con `HipotecaType` (solo los campos propios del
   tipo; partes e inmuebles ya viven en `Acto`).
2. Agregar el `xs:include` en `testimonio-digital.xsd`.
3. Agregar `<xs:element name="Hipoteca" type="HipotecaType"/>` al `xs:choice`
   de `ActoType`.
4. Los roles `ACREEDOR`/`DEUDOR` ya existen en `RolParteEnum`.
```
