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

## Sobre la firma en estos ejemplos

**Los ejemplos no llevan firma adentro, y es correcto que no la lleven.** Desde
la **v3.1** la firma del escribano es **detached**: viaja en una parte aparte
del envío (`firma_xml_escribano`) y el documento no la contiene. Un XML con un
`<ds:Signature>` adentro **lo rechaza el RPI**, así que estos ejemplos son el
XML completo tal como va en la parte `xml` del multipart.

Lo que les falta para que el RPI los acepte no es estructura, es firma: hay que
firmarlos con un certificado real del escribano autorizante y mandar esa firma
en su propia parte.

Cuando implementes el cliente:

1. Generá el XML completo (como estos ejemplos), con el `HashPDF` del PDF ya
   firmado.
2. Firmalo **en modo detached** con la librería XAdES de tu stack (ver
   [05 — Firma digital](../docs/05-firma-digital.md)). La firma cubre los
   **bytes exactos** del archivo: no lo reformatees ni lo reserialices después.
3. Mandá el XML y la firma como partes separadas del multipart (ver
   [03 — Endpoint API](../docs/03-endpoint-api.md)).

> Hasta la v3.0 estos ejemplos traían un `<ds:Signature>` con valores base64
> ficticios (`UExBQ0VIT0xERVI...` = "PLACEHOLDER..."). Se quitó: con la firma
> detached ese bloque hace que el envío se rechace.

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
