# XSD — Contratos modulares

> **El contrato vigente es la v3.0**, definida en [`v3/`](v3/) (namespace `/v3`,
> el tipo de acto es un `<Codigo>`). Si vas a integrar, validá contra
> `v3/testimonio-digital.xsd` y leé [`v3/README.md`](v3/README.md).

Esta carpeta contiene los esquemas XSD que definen el contrato del testimonio
digital. La organización es **modular**: un entry point más componentes comunes
reutilizables, todos bajo el mismo namespace.

## Organización

```
xsd/
├── xmldsig-core-schema.xsd      ← XSD oficial de W3C XML-DSig (copia local, dominio público)
└── v3/                          ← ★ contrato vigente
    ├── README.md
    ├── testimonio-digital.xsd   ← entry point (lo que el cliente valida)
    └── comunes/                 ← tipos reutilizables por todos los actos
        ├── metadatos-envio.xsd
        ├── escribano-autorizante.xsd
        ├── otorgamiento.xsd
        ├── persona.xsd
        ├── parte.xsd
        ├── certificacion-inhibicion.xsd
        ├── identificacion-inmueble.xsd
        ├── certificacion-catastral.xsd
        ├── nomenclatura-catastral.xsd
        ├── certificacion-dominio.xsd
        ├── datos-economicos.xsd
        ├── visado-rentas.xsd
        └── rogante.xsd
```

**No hay carpeta `actos/`**: el tipo de acto es un dato (`<Codigo>`), no un
elemento del esquema, así que ningún acto tiene su propio XSD.

## Sobre xmldsig-core-schema.xsd

Es una copia local del XSD oficial de W3C XML Digital Signature. Se incluye en
el paquete para que la validación funcione **offline**, sin depender de la URL
de W3C (`http://www.w3.org/TR/xmldsig-core/xmldsig-core-schema.xsd`).

El archivo está en dominio público (W3C Software License). Su contenido no es
modificable por el contrato — es el estándar W3C tal cual.

## Namespace

Todos los XSD del contrato comparten el mismo namespace:

```
https://contrato.rpi.jusneuquen.gov.ar/testimonio-digital/v3
```

Esto se logra usando `xs:include` (mismo namespace) en lugar de `xs:import`
(distinto namespace). De esta forma el cliente solo necesita validar contra
`v3/testimonio-digital.xsd` y todas las definiciones se resuelven automáticamente.

## Cómo validar

### Desde línea de comandos con xmllint

```bash
xmllint --schema v3/testimonio-digital.xsd ../ejemplos/v3/compraventa-minima.xml --noout
```

### Desde código

Cualquier parser XSD estándar puede validar contra `v3/testimonio-digital.xsd`.
El parser resuelve los `xs:include` automáticamente, siempre que se respete
la estructura de carpetas (`comunes/`).

## Cómo agregar un nuevo acto

**No se toca el XSD.** Habilitar un acto es sumar su código en el servicio (con
su familia estructural) y publicarlo en `catalogo-actos.json` y
`artefacto-campos-por-acto.json`. Ver [`v3/README.md`](v3/README.md) y el
ADR-004 del repo del servicio.

## Convenciones

- **Tipos**: nombres en PascalCase terminados en `Type` (ej: `PersonaType`).
- **Elementos**: nombres en PascalCase (ej: `EscribanoAutorizante`).
- **Atributos**: nombres en camelCase (ej: `version`, `rol`, `numero`).
- **Documentación**: cada tipo y elemento principal lleva `xs:annotation/xs:documentation`
  en español.

## Validación de las extensiones

Algunos campos tienen restricciones que el XSD no puede validar completamente:

- **Formatos de fecha**: el XSD valida `xs:date` pero no que la fecha esté en
  el pasado o sea coherente con otras fechas.
- **CUIT**: el XSD valida el formato (con o sin guiones) pero no el dígito
  verificador.
- **Hash SHA-256**: el XSD valida que sean 64 caracteres hexadecimales pero
  no que coincida con el PDF.
- **Código de acto**: el XSD valida que sea un entero, pero no que ese código
  esté **habilitado** ni qué bloques exige.

Estas validaciones adicionales las hace el servicio del RPI después de la
validación XSD.
