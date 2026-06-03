# XSD — Contratos modulares

Esta carpeta contiene los esquemas XSD que definen el contrato del testimonio
digital. La organización es **modular**: hay componentes comunes reutilizables
y específicos por tipo de acto.

## Organización

```
xsd/
├── testimonio-digital.xsd       ← entry point (lo que el cliente valida)
├── xmldsig-core-schema.xsd      ← XSD oficial de W3C XML-DSig (copia local, dominio público)
├── comunes/                     ← tipos reutilizables entre actos
│   ├── metadatos-envio.xsd
│   ├── escribano-autorizante.xsd
│   ├── persona.xsd
│   ├── identificacion-inmueble.xsd
│   ├── datos-economicos.xsd
│   ├── certificacion-registral.xsd
│   ├── certificacion-catastral.xsd
│   ├── nomenclatura-catastral.xsd
│   ├── visado-rentas.xsd
│   ├── otorgamiento.xsd
│   └── rogante.xsd
└── actos/
    └── compraventa.xsd          ← en v1.0 solo compraventa
```

## Sobre xmldsig-core-schema.xsd

Es una copia local del XSD oficial de W3C XML Digital Signature. Se incluye en
el paquete para que la validación funcione **offline**, sin depender de la URL
de W3C (`http://www.w3.org/TR/xmldsig-core/xmldsig-core-schema.xsd`).

El archivo está en dominio público (W3C Software License). Su contenido no es
modificable por el contrato — es el estándar W3C tal cual.

## Namespace

Todos los XSD comparten el mismo namespace:

```
https://contrato.rpi.jusneuquen.gov.ar/testimonio-digital/v1
```

Esto se logra usando `xs:include` (mismo namespace) en lugar de `xs:import`
(distinto namespace). De esta forma el cliente solo necesita validar contra
`testimonio-digital.xsd` y todas las definiciones se resuelven automáticamente.

## Cómo validar

### Desde línea de comandos con xmllint

```bash
xmllint --schema testimonio-digital.xsd ../ejemplos/compraventa-minima.xml --noout
```

### Desde código

Cualquier parser XSD estándar puede validar contra `testimonio-digital.xsd`.
El parser resuelve los `xs:include` automáticamente, siempre que se respete
la estructura de carpetas (`comunes/`, `actos/`).

## Cómo agregar un nuevo acto en futuras versiones

Para agregar, por ejemplo, hipoteca en v2.0:

1. Crear `actos/hipoteca.xsd` con el tipo `HipotecaType`.
2. Agregar el include en `testimonio-digital.xsd`.
3. Agregar la opción al `xs:choice` del elemento `Acto`.
4. Los tipos comunes (`Persona`, `IdentificacionInmueble`, etc.) se
   reutilizan sin cambios.

## Convenciones

- **Tipos**: nombres en PascalCase terminados en `Type` (ej: `PersonaType`).
- **Elementos**: nombres en PascalCase (ej: `Adquirentes`, `EscribanoAutorizante`).
- **Atributos**: nombres en camelCase (ej: `version`).
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

Estas validaciones adicionales las hace el servicio del RPI después de la
validación XSD.
