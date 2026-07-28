# XSD v3 — Testimonio Digital (el tipo de acto es un CÓDIGO)

v3 del contrato. Coexiste con las versiones previas en disco: v1 en `xsd/`, v2 en
`xsd/v2/` (congelado). v3 vive en este `xsd/v3/` con su propio namespace.

## Namespace

```
https://contrato.rpi.jusneuquen.gov.ar/testimonio-digital/v3
```

Sufijo `/v3`, consistente en todos los XSD de v3. Todos los componentes se enlazan
con `xs:include` (mismo namespace); el cliente valida contra
`xsd/v3/testimonio-digital.xsd` y los includes se resuelven solos.

## Qué cambia respecto de v2 (MAJOR, incompatible)

**El tipo de acto pasa de ser un elemento del esquema a ser un DATO.** En v2 cada
tipo era un elemento del `xs:choice` de `ActoType` (`<Compraventa/>`, `<Hipoteca/>`,
`<Donacion/>`, `<Permuta/>`) con un `xs:complexType` propio. Esos tipos estaban
**vacíos** (solo textos libres): la sustancia del acto ya vivía genérica a nivel
`<Acto>`. v3 elimina el choice y modela el acto con **`<Codigo>`**, el código del
catálogo legacy `act`. Ver ADR-004 (repo del servicio).

Cambios concretos respecto de v2:

- **`<Codigo>` (xs:integer)** reemplaza al `xs:choice`. **NO es un enum** de los ~200
  códigos: el catálogo cambia y no debe versionar el contrato. La lista de códigos
  existentes se publica como dato en `catalogo-actos.json` (raíz del repo); qué
  códigos están **habilitados** lo decide el servicio, no el XSD.
- **Los 4 XSD `actos/` se eliminan.** Ya no hay `actos/` en v3.
- **Los textos libres suben a `<Acto>`** como opcionales (antes vivían en los
  `XxxType`): `DescripcionActoIncompleto`, `ReconocimientoHipotecaMedidasCautelares`,
  `AfectacionesAlDominio`, `AsentimientoConyugal`.
- **Catastro opcional**: `CertificacionCatastral` y `NomenclaturaCatastral` pasan a
  `minOccurs=0` (eran obligatorias por inmueble). La familia HIPOTECARIA-LIBERA
  (cancelación, liberación) no las lleva; su obligatoriedad para las familias que sí
  las requieren se valida en el servicio (`exigeCatastral`). Es la **única**
  obligatoriedad estructural que v3 relaja.
- **`<ActoSecundario>`** (código 1075/1157, desde v2.4.0) **no cambia**.

Todo lo demás es igual a v2: N actos, partes con rol genérico, Inmuebles /
DatosEconomicos / VisadoRentas obligatorios, componentes comunes.

## Estructura

```
TestimonioDigital (version="3.0")
├── MetadatosEnvio
├── EscribanoAutorizante
├── Otorgamiento
├── Actos
│   └── Acto (numero, 1..N)
│       ├── Codigo            (xs:integer — código del catálogo `act`)   ← acto PRINCIPAL
│       ├── ActoSecundario?   (código 1075|1157)                         ← acto SECUNDARIO (opcional)
│       ├── Partes
│       │   └── Parte (rol, 1..N)
│       │       ├── …campos de PersonaType (Tipo … PEP … Proporcion, Representante)
│       │       └── CertificacionInhibicion   (opcional; obligatoria por regla según familia)
│       ├── Inmuebles
│       │   └── Inmueble (1..M)
│       │       ├── IdentificacionInmueble
│       │       ├── CertificacionCatastral    (OPCIONAL en v3)
│       │       └── NomenclaturaCatastral      (OPCIONAL en v3)
│       ├── DatosEconomicos
│       ├── CertificacionDominio               (opcional XSD; obligatoria por regla según familia)
│       ├── VisadoRentas
│       └── (textos libres opcionales)         DescripcionActoIncompleto, Reconocimiento…, Afectaciones…, AsentimientoConyugal
├── Rogante
├── TextoCuerpo
├── Observaciones (opcional)
└── ds:Signature
```

```
xsd/v3/
├── testimonio-digital.xsd          ← entry point (ActoType con <Codigo>, Actos, Inmueble, raíz)
└── comunes/                        ← tipos comunes (SIN actos/: el acto no tiene tipo XSD propio)
    ├── parte.xsd                   ← ParteType (= PersonaType + rol + CertificacionInhibicion)
    ├── persona.xsd                 ← PersonaType (con Proporcion/Representante)
    ├── identificacion-inmueble.xsd
    ├── certificacion-catastral.xsd ← (referenciado en Inmueble, opcional en v3)
    ├── nomenclatura-catastral.xsd  ← (referenciado en Inmueble, opcional en v3)
    ├── certificacion-dominio.xsd
    ├── certificacion-inhibicion.xsd
    └── …                           ← datos-economicos, visado-rentas, metadatos-envio, …
```

`xmldsig-core-schema.xsd` se reutiliza por `xs:import` desde `../xmldsig-core-schema.xsd`
(compartido con v1/v2, estándar W3C, dominio público).

## Modelo de partes (rol genérico)

`ParteType` extiende `PersonaType` y agrega el atributo obligatorio `rol`
(`RolParteEnum`) más el elemento opcional `CertificacionInhibicion` (último hijo).
Roles definidos: `ADQUIRENTE`, `TRANSMITENTE`, `ACREEDOR`, `DEUDOR`. Agregar un rol =
una línea `<xs:enumeration>` en `parte.xsd`.

## Reglas que el XSD NO valida (van en el servicio, coherente con ADR-002/ADR-004)

El XSD v3 es deliberadamente permisivo. Se validan en el servicio, por combinación
**código → familia + override**:

- **Qué código está HABILITADO** (`CODIGO_ACTO_NO_HABILITADO` si no lo está).
- **Qué roles admite el acto**, cuáles son obligatorios, cuáles admiten proporción y
  que las proporciones sumen 1.
- **Qué montos exige** (precio/valuación/`MontoHipoteca`) según la familia.
- **Qué certificaciones exige** (`exigeDominio`, `exigeCatastral`, inhibición por rol).
- Unicidad de `numero`; validaciones condicionales heredadas (CUIT, fechas, etc.).

## Cómo validar

```bash
xmllint --schema xsd/v3/testimonio-digital.xsd \
  ejemplos/v3/compraventa-minima.xml --noout
```

Ejemplos destacados en `ejemplos/v3/`:
- `compraventa-*.xml` — `<Codigo>1028</Codigo>` en sus variantes (mínima, USD, jurídica,
  representante, inmueble antiguo, múltiples titulares, dos actos).
- `hipoteca-ejemplo-valido.xml` — `<Codigo>1075</Codigo>` (ACREEDOR/DEUDOR, `MontoHipoteca`).
- `donacion-ejemplo-valido.xml` — `<Codigo>1056</Codigo>`.
- `permuta-ejemplo-valido.xml` — `<Codigo>1102</Codigo>`, permuta de 2 actos con cruce.
- `compraventa-con-hipoteca-ejemplo-valido.xml` — `<Codigo>1028</Codigo>` +
  `<ActoSecundario>1075</ActoSecundario>` (una minuta, dos actos).
- `cancelacion-hipoteca-ejemplo-valido.xml` — `<Codigo>1020</Codigo>`, familia
  HIPOTECARIA-LIBERA. **SIN catastro, dominio ni inhibiciones**: es el caso que v2 NO
  podía expresar (catastro obligatorio) y v3 habilita.

## Acto secundario (`<ActoSecundario>`)

Es la dimensión "acto principal + secundario" del legacy (`wmmac1`/`wmmac2`). En v3
**ambos son códigos**: el principal en `<Codigo>` y el secundario (restringido a
{1075, 1157}) en el `<ActoSecundario>` opcional, inmediatamente después. No agrega
estructura: partes, inmuebles y montos son compartidos por ambos actos del `<Acto>`.

## Agregar un acto nuevo — ya NO se toca el XSD

Esta es la diferencia central de v3. En v2, agregar un acto era crear un `actos/*.xsd`,
un `xs:include` y una línea en el `xs:choice`. **En v3 el contrato no cambia.** Un acto
nuevo es un **código más** que el XSD ya acepta (cualquier `xs:integer`).

Habilitar un acto es trabajo del **servicio** (rpi-td), no del contrato:

1. Sumar el código a `CODIGOS_HABILITADOS` y darle su entrada en el registro
   (familia estructural + override si diverge). TypeScript obliga a completarla.
2. **Verificar la familia/override contra filas reales de producción** — es el costo
   que v3 NO elimina (ver ⚠️ en ADR-004). El contrato ya no cambia, pero saber a qué
   familia pertenece un acto y si tiene override sigue requiriendo mirar datos reales.
3. Agregar el código a `catalogo-actos.json` (dato informativo) y, si querés, un
   ejemplo en `ejemplos/v3/`.

El XSD solo se tocaría si apareciera una **estructura nueva** universal (un bloque que
hoy no existe en `<Acto>`), no por agregar un acto.
