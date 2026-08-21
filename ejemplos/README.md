# Ejemplos de Testimonio Digital

Esta carpeta contiene XMLs de ejemplo válidos contra el contrato XSD. Sirven
como referencia mientras el programador construye su propio generador.

Los ejemplos vigentes están en [`v3/`](v3/) y validan contra
`xsd/v3/testimonio-digital.xsd`.

## Archivos (en `v3/`)

| Archivo | Código | Descripción |
|---------|--------|-------------|
| `compraventa-minima.xml` | 1028 | Caso más simple: 1 acto con 1 parte ADQUIRENTE humana soltera, 1 TRANSMITENTE humano, 1 inmueble con matrícula, monto en pesos, visado de Rentas R, certificación catastral emitida sin observaciones. |
| `compraventa-multiple-titulares.xml` | 1028 | 1 acto con 2 partes ADQUIRENTE casadas al 50% cada una, 2 TRANSMITENTE (matrimonio), 1 inmueble, con nomenclatura catastral y asentimiento conyugal. |
| `compraventa-usd.xml` | 1028 | Compraventa en dólares con cotización, visado de Rentas A (sin visado), certificación catastral no emitida. |
| `compraventa-persona-juridica.xml` | 1028 | Parte ADQUIRENTE persona jurídica (S.A.) con `Tipo=J` e inscripción ante organismo; TRANSMITENTE humano. |
| `compraventa-con-representante.xml` | 1028 | Parte ADQUIRENTE humana menor de edad con bloque `Representante` (tutor); TRANSMITENTE humano. |
| `compraventa-inmueble-antiguo.xml` | 1028 | Inmueble previo a la matriculación: sin `Matricula`, identificado por `Tomo`/`Folio`/`Finca`. |
| `compraventa-dos-actos.xml` | 1028 + 1028 | Testimonio con **2 actos** (`<Acto numero="1">` y `<Acto numero="2">`): personas humanas y jurídica con representante, monto en $ y en USD. |
| `compraventa-con-hipoteca-ejemplo-valido.xml` | 1028 + `<ActoSecundario>1075` | Una minuta con acto principal y secundario. |
| `hipoteca-ejemplo-valido.xml` | 1075 | Roles ACREEDOR / DEUDOR, con `MontoHipoteca`. |
| `donacion-ejemplo-valido.xml` | 1056 | Donación. |
| `permuta-ejemplo-valido.xml` | 1102 + 1102 | Permuta de 2 actos con cruce de partes. |
| `cancelacion-hipoteca-ejemplo-valido.xml` | 1020 | Familia HIPOTECARIA-LIBERA: **sin catastro, dominio ni inhibiciones**. |

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

Para validar la estructura contra el XSD:

```bash
xmllint --schema ../xsd/v3/testimonio-digital.xsd v3/compraventa-minima.xml --noout
```

Debería responder:

```
v3/compraventa-minima.xml validates
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
