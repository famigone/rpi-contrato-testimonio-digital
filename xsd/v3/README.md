# XSD v3 — Testimonio Digital (el tipo de acto es un CÓDIGO)

Contrato XSD vigente del testimonio digital.

## Namespace

```
https://contrato.rpi.jusneuquen.gov.ar/testimonio-digital/v3
```

Sufijo `/v3`, consistente en todos los XSD. Todos los componentes se enlazan
con `xs:include` (mismo namespace); el cliente valida contra
`xsd/v3/testimonio-digital.xsd` y los includes se resuelven solos.

## El tipo de acto es un DATO

**El acto se modela con `<Codigo>`**, el código del catálogo `act`, no con un
elemento nombrado por tipo. Ver ADR-004 (repo del servicio). Consecuencias:

- **`<Codigo>` es un `xs:integer`** (hasta 4 dígitos) y **NO un enum** de los ~200
  códigos: el catálogo cambia y no debe versionar el contrato. La lista de códigos
  existentes se publica como dato en `catalogo-actos.json` (raíz del repo); qué
  códigos están **habilitados** lo decide el servicio, no el XSD.
- **No hay `actos/`**: ningún acto tiene un XSD propio.
- **Los textos libres son hijos opcionales de `<Acto>`**, al final:
  `DescripcionActoIncompleto`, `ReconocimientoHipotecaMedidasCautelares`,
  `AfectacionesAlDominio`, `AsentimientoConyugal`.
- **Catastro opcional**: `CertificacionCatastral` y `NomenclaturaCatastral` son
  `minOccurs=0`. La familia HIPOTECARIA-LIBERA (cancelación, liberación) no las
  lleva; para las familias que sí las requieren, la obligatoriedad la valida el
  servicio (`exigeCatastral`).
- **`<ActoSecundario>`** (código 1075/1157) es opcional y va inmediatamente
  después de `<Codigo>`.

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
│       │       ├── CertificacionCatastral    (OPCIONAL)
│       │       └── NomenclaturaCatastral      (OPCIONAL)
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
    ├── certificacion-catastral.xsd ← (referenciado en Inmueble, opcional)
    ├── nomenclatura-catastral.xsd  ← (referenciado en Inmueble, opcional)
    ├── certificacion-dominio.xsd
    ├── certificacion-inhibicion.xsd
    └── …                           ← datos-economicos, visado-rentas, metadatos-envio, …
```

`xmldsig-core-schema.xsd` se reutiliza por `xs:import` desde `../xmldsig-core-schema.xsd`
(estándar W3C, dominio público).

## Modelo de partes (rol genérico)

`ParteType` extiende `PersonaType` y agrega el atributo obligatorio `rol`
(`RolParteEnum`) más el elemento opcional `CertificacionInhibicion` (último hijo).
Roles definidos: `ADQUIRENTE`, `TRANSMITENTE`, `ACREEDOR`, `DEUDOR`. Agregar un rol =
una línea `<xs:enumeration>` en `parte.xsd`.

## Reglas que el XSD NO valida (van en el servicio, coherente con ADR-002/ADR-004)

El XSD es deliberadamente permisivo. Se validan en el servicio, por combinación
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
  HIPOTECARIA-LIBERA. **SIN catastro, dominio ni inhibiciones**.

## Acto secundario (`<ActoSecundario>`)

Es la dimensión "acto principal + secundario" del legacy (`wmmac1`/`wmmac2`).
**Ambos son códigos**: el principal en `<Codigo>` y el secundario (restringido a
{1075, 1157}) en el `<ActoSecundario>` opcional, inmediatamente después. No agrega
estructura: partes, inmuebles y montos son compartidos por ambos actos del `<Acto>`.

## Agregar un acto nuevo — no se toca el XSD

**El contrato no cambia al habilitar un acto.** Un acto nuevo es un **código más**
que el XSD ya acepta (cualquier `xs:integer`).

Habilitar un acto es trabajo del **servicio** (rpi-td), no del contrato:

1. Sumar el código a `CODIGOS_HABILITADOS` y darle su entrada en el registro
   (familia estructural + override si diverge). TypeScript obliga a completarla.
2. **Verificar la familia/override contra filas reales de producción** — es el costo
   que el contrato NO elimina (ver ⚠️ en ADR-004): saber a qué familia pertenece un
   acto y si tiene override sigue requiriendo mirar datos reales.
3. Agregar el código a `catalogo-actos.json` (dato informativo) y, si querés, un
   ejemplo en `ejemplos/v3/`.

El XSD solo se tocaría si apareciera una **estructura nueva** universal (un bloque que
hoy no existe en `<Acto>`), no por agregar un acto.
