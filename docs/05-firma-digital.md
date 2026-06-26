# 05 — Firma digital

## Estándar de firma

El XML del testimonio debe estar firmado con **XML Digital Signature (XML-DSig)**
siguiendo el estándar [W3C Recommendation](https://www.w3.org/TR/xmldsig-core/).

El bloque de firma va dentro del documento, como último hijo del elemento raíz
`TestimonioDigital`, usando el namespace `http://www.w3.org/2000/09/xmldsig#`.

## Tipo de firma

Se usa una firma **enveloped**: la firma está embebida dentro del mismo documento
XML que firma. Esto implica que la transformación `enveloped-signature` debe
estar presente para que el verificador excluya el bloque `Signature` al
recalcular el hash.

## Algoritmos requeridos

| Componente | Algoritmo |
|------------|-----------|
| Canonicalización | `http://www.w3.org/TR/2001/REC-xml-c14n-20010315` (Canonical XML 1.0) |
| Algoritmo de firma | `http://www.w3.org/2001/04/xmldsig-more#rsa-sha256` (RSA-SHA256) |
| Digest del referencia | `http://www.w3.org/2001/04/xmlenc#sha256` (SHA-256) |
| Transformación | `http://www.w3.org/2000/09/xmldsig#enveloped-signature` |

## Certificado del firmante

> ⚠️ **Pendiente de confirmación**. Los detalles específicos sobre la cadena
> de certificación válida en el ámbito del Poder Judicial de Neuquén están
> en proceso de definición. Se confirmará antes de homologación.

En principio, el certificado del firmante debe cumplir:

- Ser un certificado válido de **firma digital** (no solo certificado SSL).
- Estar emitido por una **autoridad certificante reconocida** por el Poder
  Judicial de Neuquén o la ONTI (Oficina Nacional de Tecnologías de la
  Información).
- Estar vigente al momento del envío (no vencido, no revocado).
- Pertenecer al **escribano autorizante** declarado en el XML (el RPI verificará
  que el CUIT del certificado coincida con el `EscribanoAutorizante/CUIT` del XML).

El certificado del firmante se incluye en el bloque `<KeyInfo><X509Data><X509Certificate>`
de la firma para que el RPI pueda validarlo sin necesidad de tener un catálogo
local de certificados de escribanos.

## Estructura de la firma

Ejemplo de bloque `<Signature>` esperado:

```xml
<ds:Signature xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
  <ds:SignedInfo>
    <ds:CanonicalizationMethod
        Algorithm="http://www.w3.org/TR/2001/REC-xml-c14n-20010315"/>
    <ds:SignatureMethod
        Algorithm="http://www.w3.org/2001/04/xmldsig-more#rsa-sha256"/>
    <ds:Reference URI="">
      <ds:Transforms>
        <ds:Transform
            Algorithm="http://www.w3.org/2000/09/xmldsig#enveloped-signature"/>
        <ds:Transform
            Algorithm="http://www.w3.org/TR/2001/REC-xml-c14n-20010315"/>
      </ds:Transforms>
      <ds:DigestMethod
          Algorithm="http://www.w3.org/2001/04/xmlenc#sha256"/>
      <ds:DigestValue>BASE64_DEL_HASH_DEL_XML_CANONICALIZADO</ds:DigestValue>
    </ds:Reference>
  </ds:SignedInfo>
  <ds:SignatureValue>BASE64_DE_LA_FIRMA_RSA</ds:SignatureValue>
  <ds:KeyInfo>
    <ds:X509Data>
      <ds:X509Certificate>BASE64_DEL_CERTIFICADO_X509_DEL_ESCRIBANO</ds:X509Certificate>
    </ds:X509Data>
  </ds:KeyInfo>
</ds:Signature>
```

El `URI=""` en `Reference` indica que se firma el documento completo
(con la transformación `enveloped-signature` excluyendo la propia firma).

## Compatibilidad con firmadores que agregan `Id`

Algunas librerías de firma XML-DSig (`xml-crypto` de Node.js, Apache
Santuario, etc.) agregan automáticamente un atributo `Id` al elemento
raíz `<TestimonioDigital>` cuando firman, para que la firma pueda
referenciarlo internamente.

El contrato acepta este atributo como opcional. Tanto

```xml
<TestimonioDigital version="2.0" xmlns="...">
```

como

```xml
<TestimonioDigital version="2.0" Id="_0" xmlns="...">
```

son válidos. El RPI ignora el contenido del atributo `Id`; lo acepta solo
para compatibilidad con firmadores que lo generan.

Si la librería que usás no agrega `Id`, no es necesario incluirlo
manualmente.

## Verificación que hace el RPI

Cuando el RPI recibe el testimonio:

1. Extrae el bloque `<Signature>` del XML.
2. Canonicaliza el resto del documento.
3. Calcula el hash SHA-256 de la canonicalización.
4. Verifica que el hash calculado coincide con el `<DigestValue>`.
5. Extrae el certificado X.509 del bloque `<KeyInfo>`.
6. Verifica que el certificado esté vigente, no revocado, y emitido por una
   autoridad reconocida.
7. Verifica que la `<SignatureValue>` sea válida para los `<SignedInfo>` con
   la clave pública del certificado.
8. Verifica que el CUIT del titular del certificado coincida con
   `EscribanoAutorizante/CUIT` del XML.

Si cualquiera de estos pasos falla, el RPI rechaza el testimonio con HTTP 401
(ver [07 — Respuestas y errores](07-respuestas-y-errores.md)).

## Bibliotecas recomendadas

Para implementación del lado del sistema del Colegio, las siguientes bibliotecas
están bien probadas:

- **Java**: Apache Santuario (`xml-security`).
- **C# / .NET**: `System.Security.Cryptography.Xml.SignedXml`.
- **Python**: `signxml`.
- **Node.js**: `xml-crypto`.
- **PHP**: `php-xmlseclibs`.

## Errores comunes a evitar

- **No incluir el certificado en `<KeyInfo>`**: si el RPI no recibe el
  certificado en el XML, no puede validar la firma. **Siempre incluirlo**.
- **Olvidar la transformación enveloped-signature**: sin ella, el hash incluye
  la propia firma y no cierra. La firma siempre fallaría.
- **Canonicalización inconsistente**: usar exactamente `c14n-20010315`. Otras
  canonicalizaciones (Exclusive C14N, C14N 1.1) no son compatibles con esta
  versión del contrato.
- **Cambiar el XML después de firmar**: cualquier cambio (incluso reformatear
  espacios en blanco fuera de elementos textuales) puede invalidar la firma.
  El XML que se envía debe ser **exactamente** el que se firmó.

## Nota sobre verificación de certificado en v2.0

La implementación actual del servicio del RPI verifica criptográficamente
la firma XML-DSig (canonicalización, hash, RSA) pero **no verifica** los
siguientes aspectos del certificado del firmante:

- Habilitación de la autoridad certificante por ONTI.
- Vigencia del certificado al momento del envío.
- Estado de revocación (CRL/OCSP).

Esta limitación está documentada como deuda técnica del servicio y será
resuelta antes de la publicación oficial de `v2.0`. El contrato describe
el comportamiento objetivo (verificación completa); la implementación se
alineará antes de salir a producción.

Para clientes implementadores: asumir que en producción el RPI verificará
los tres aspectos mencionados. Asegurar que el certificado del escribano:

- Sea emitido por una autoridad certificante habilitada por ONTI.
- Esté vigente al momento de cada envío (renovar antes del vencimiento).
- No esté revocado.

Si alguno de estos checks falla en el futuro, el RPI rechazará el
testimonio con un código de error específico (`CERTIFICADO_NO_VIGENTE`,
`CERTIFICADO_REVOCADO`, `CERTIFICADO_NO_RECONOCIDO`) que ya están
documentados en `07-respuestas-y-errores.md`.

## Próximos pasos

- Para el adjunto PDF y su verificación, andá a [06 — Adjunto PDF](06-adjunto-pdf.md).
- Para los códigos de error de firma, andá a [07 — Respuestas y errores](07-respuestas-y-errores.md).
