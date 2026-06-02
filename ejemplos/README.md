# Ejemplos de Testimonio Digital

Esta carpeta contiene XMLs de ejemplo válidos contra el contrato XSD. Sirven
como referencia mientras el programador construye su propio generador.

## Archivos

| Archivo | Descripción |
|---------|-------------|
| `compraventa-minima.xml` | Caso más simple: 1 adquirente, 1 transmitente, 1 inmueble, monto en pesos. |
| `compraventa-multiple-titulares.xml` | 2 adquirentes con proporción 1/2 cada uno, 2 transmitentes (matrimonio), 1 inmueble. |
| `compraventa-usd.xml` | Compraventa en dólares estadounidenses con cotización. |

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
- Matrículas: del rango ficticio 12-3456 a 12-3499.
- Montos: arbitrarios.

No corresponden a personas o inmuebles reales.
