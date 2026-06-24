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
- **Bajan del testimonio al acto**: `DatosEconomicos`, `CertificacionCatastral`,
  `NomenclaturaCatastral` (opcional), `CertificacionRegistralPrevia`,
  `VisadoRentas`, además de `Partes` e `Inmuebles`.
- **Quedan a nivel testimonio** (un trámite / un cuerpo de escritura, igual que
  v1): `MetadatosEnvio`, `EscribanoAutorizante`, `Otorgamiento`, `Rogante`,
  `TextoCuerpo`, `Observaciones`, `Signature`.
- **Partes con rol genérico**: se reemplazan los contenedores `Adquirentes` /
  `Transmitentes` de v1 por una lista de `<Parte rol="...">`. Ver `comunes/parte.xsd`.

## Estructura

```
TestimonioDigital (version="2.0")
├── MetadatosEnvio
├── EscribanoAutorizante
├── Otorgamiento
├── Actos
│   └── Acto (numero, 1..N)
│       ├── (choice)  Compraventa | …futuros…
│       ├── Partes → Parte (rol, 1..N)
│       ├── Inmuebles → Inmueble (1..M)
│       ├── DatosEconomicos
│       ├── CertificacionCatastral
│       ├── NomenclaturaCatastral (opcional)
│       ├── CertificacionRegistralPrevia
│       └── VisadoRentas
├── Rogante
├── TextoCuerpo
├── Observaciones (opcional)
└── ds:Signature
```

```
xsd/v2/
├── testimonio-digital.xsd      ← entry point (define ActoType, Actos, raíz)
├── comunes/
│   ├── parte.xsd               ← NUEVO: ParteType (= PersonaType + rol) y RolParteEnum
│   ├── persona.xsd             ← reutilizado de v1 (PersonaType con Proporcion/Representante)
│   └── …                       ← resto de comunes, reutilizados de v1
└── actos/
    └── compraventa.xsd         ← CompraventaType SIN partes/inmuebles (viven en Acto)
```

`xmldsig-core-schema.xsd` se reutiliza por `xs:import` desde la copia compartida
en `../xmldsig-core-schema.xsd` (estándar W3C, dominio público).

## Modelo de partes (rol genérico)

`ParteType` **extiende** `PersonaType` y agrega el atributo obligatorio `rol`
(`RolParteEnum`). No se rehace la persona: una `<Parte>` tiene exactamente el
contenido de `PersonaType` (incluida `Proporcion` y `Representante`) más `rol`.

Roles definidos hoy: `ADQUIRENTE`, `TRANSMITENTE`, `ACREEDOR`, `DEUDOR`
(acreedor/deudor quedan listos aunque Hipoteca aún no esté en el choice).
Agregar un rol = una línea `<xs:enumeration>` en `parte.xsd`.

## Reglas que el XSD NO valida (van en código, coherente con ADR-002)

Intencional y documentado en los propios XSD:

- **Unicidad de `numero`** entre actos.
- **Qué roles corresponden a qué tipo de acto** (el XSD acepta cualquier
  `RolParte` en cualquier acto).
- **Qué roles admiten proporción** y que las **proporciones sumen 1 por acto**.
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
