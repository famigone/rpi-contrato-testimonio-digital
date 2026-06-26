# Ejemplos de Testimonio Digital

Esta carpeta contiene XMLs de ejemplo válidos contra el contrato XSD. Sirven
como referencia mientras el programador construye su propio generador.

Los ejemplos **vigentes (v2)** están en [`v2/`](v2/) y validan contra
`xsd/v2/testimonio-digital.xsd`. Los archivos `*.xml` de esta carpeta raíz son
los ejemplos **v1 (legacy)**, que validan contra `xsd/testimonio-digital.xsd` y
se conservan porque v1 coexiste con v2.

## Archivos (v2 — en `v2/`)

| Archivo | Descripción |
|---------|-------------|
| `compraventa-minima.xml` | Caso más simple: 1 acto con 1 parte ADQUIRENTE humana soltera, 1 TRANSMITENTE humano, 1 inmueble con matrícula, monto en pesos, visado de Rentas R, certificación catastral emitida sin observaciones. |
| `compraventa-multiple-titulares.xml` | 1 acto con 2 partes ADQUIRENTE casadas al 50% cada una, 2 TRANSMITENTE (matrimonio), 1 inmueble, con nomenclatura catastral y asentimiento conyugal. |
| `compraventa-usd.xml` | Compraventa en dólares con cotización, visado de Rentas A (sin visado), certificación catastral no emitida. |
| `compraventa-persona-juridica.xml` | Parte ADQUIRENTE persona jurídica (S.A.) con `Tipo=J` e inscripción ante organismo; TRANSMITENTE humano. |
| `compraventa-con-representante.xml` | Parte ADQUIRENTE humana menor de edad con bloque `Representante` (tutor); TRANSMITENTE humano. |
| `compraventa-inmueble-antiguo.xml` | Inmueble previo a la matriculación: sin `Matricula`, identificado por `Tomo`/`Folio`/`Finca`. |
| `compraventa-dos-actos.xml` | Testimonio con **2 actos** (`<Acto numero="1">` y `<Acto numero="2">`): personas humanas y jurídica con representante, monto en $ y en USD. Muestra la novedad central de v2. |

## Sobre la firma XML-DSig en estos ejemplos

Los ejemplos incluyen un bloque `<ds:Signature>` con **valores base64 válidos
pero ficticios** (strings que empiezan con `UExBQ0VIT0xERVI...` que decodificado
significa "PLACEHOLDER..."). Esto permite que los ejemplos validen contra el XSD,
pero **no pasarán la verificación criptográfica del RPI**.

Sirven solo como referencia de estructura. Para producir un XML que el RPI
acepte, hay que firmarlo con un certificado real del escribano autorizante.

Cuando implementes el cliente, deberás:

1. Generar el XML sin el bloque Signature.
2. Firmarlo con la librería XML-DSig de tu stack (ver
   [05 — Firma digital](../docs/05-firma-digital.md)).
3. La librería inserta el bloque `<ds:Signature>` con valores reales.

## Validación de los ejemplos

Para validar la estructura contra el XSD v2:

```bash
xmllint --schema ../xsd/v2/testimonio-digital.xsd v2/compraventa-minima.xml --noout
```

Debería responder:

```
v2/compraventa-minima.xml validates
```

Si hay un error de validación, indica qué elemento o atributo no cumple.

## Sobre los datos en los ejemplos

Todos los datos son **ficticios**:

- CUITs, DNIs, nombres: inventados.
- Matrículas: del rango ficticio 3456 a 3499.
- Montos: arbitrarios.

No corresponden a personas o inmuebles reales.

Todos los ejemplos incluyen un `<TextoCuerpo>` con texto notarial realista
pero ficticio. En testimonios reales, el `TextoCuerpo` contiene la
transcripción completa del cuerpo de la escritura, no un resumen.
