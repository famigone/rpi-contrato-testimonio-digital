# Ejemplos de Testimonio Digital

Esta carpeta contiene XMLs de ejemplo válidos contra el contrato XSD. Sirven
como referencia mientras el programador construye su propio generador.

## Archivos

| Archivo | Descripción |
|---------|-------------|
| `compraventa-minima.xml` | Caso más simple: 1 adquirente humano soltero, 1 transmitente humano, 1 inmueble con matrícula, monto en pesos, visado de Rentas R, certificación catastral emitida sin observaciones. |
| `compraventa-multiple-titulares.xml` | 2 adquirentes humanos casados al 50% cada uno, 2 transmitentes (matrimonio), 1 inmueble, con nomenclatura catastral y asentimiento conyugal. |
| `compraventa-usd.xml` | Compraventa en dólares con cotización, visado de Rentas A (sin visado), certificación catastral no emitida. |
| `compraventa-persona-juridica.xml` | Adquirente persona jurídica (S.A.) con `Tipo=J` e inscripción ante organismo; transmitente humano. |
| `compraventa-con-representante.xml` | Adquirente humano menor de edad con bloque `Representante` (tutor); transmitente humano. |
| `compraventa-inmueble-antiguo.xml` | Inmueble previo a la matriculación: sin `Matricula`, identificado por `Tomo`/`Folio`/`Finca`. |

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
xmllint --schema ../xsd/testimonio-digital.xsd compraventa-minima.xml --noout
```

Debería responder:

```
compraventa-minima.xml validates
```

Si hay un error de validación, indica qué elemento o atributo no cumple.

## Sobre los datos en los ejemplos

Todos los datos son **ficticios**:

- CUITs, DNIs, nombres: inventados.
- Matrículas: del rango ficticio 3456 a 3499.
- Montos: arbitrarios.

No corresponden a personas o inmuebles reales.
