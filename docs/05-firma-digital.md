# 05 — Firma digital

## Estándar de firma

El XML del testimonio debe estar firmado con **XML Digital Signature (XML-DSig)**
siguiendo el estándar [W3C Recommendation](https://www.w3.org/TR/xmldsig-core/),
en perfil **XAdES-BES** ([ETSI TS 101 903](https://uri.etsi.org/01903/v1.3.2#)).

## Tipo de firma: detached

Desde la **v3.1** del contrato, la firma es **detached**: viaja como un archivo
aparte y **no** va dentro del documento XML. El XML que se envía es exactamente
el que firmó el escribano, sin un bloque `<ds:Signature>` adentro.

La firma se manda como una parte más del multipart, llamada
**`firma_xml_escribano`** (ver [03 — Endpoint API](03-endpoint-api.md)).

> ⚠️ **La firma es obligatoria.** Desde v3.1 el XSD declara `<ds:Signature>` con
> `minOccurs="0"`, porque en detached el documento no la contiene. Eso relaja el
> *esquema*, no la *regla*: el RPI rechaza todo envío que no traiga la parte
> `firma_xml_escribano`.

> ⚠️ **No se acepta la firma envuelta (enveloped).** Si el XML trae un
> `<ds:Signature>` adentro, el RPI lo **rechaza**, incluso si esa firma fuera
> criptográficamente válida. Un solo modo de firma, sin ambigüedad sobre qué
> bytes se verifican.

## Qué se firma exactamente

La firma lleva **dos referencias**:

| Referencia | `URI` | Qué cubre |
|------------|-------|-----------|
| Documento | el nombre del archivo XML (p. ej. `minuta_id_691.xml`) | Los **bytes crudos** del XML |
| SignedProperties | `#<Id de SignedProperties>` | Las propiedades firmadas XAdES, canonicalizadas |

### El digest del documento es sobre los bytes exactos

La referencia al documento **no lleva `<ds:Transforms>`**. Por el estándar, eso
significa que el digest se calcula sobre los **octetos tal como están**, sin
canonicalizar. En la práctica:

- **Los finales de línea son parte de lo firmado.** Un archivo firmado con CRLF
  (`\r\n`) y reenviado con LF (`\n`) tiene otro digest y la firma falla. Es el
  caso más frecuente, porque muchas herramientas normalizan los saltos de línea
  al leer o escribir el archivo.
- Lo mismo vale para el BOM, la indentación, los espacios entre atributos y el
  salto de línea final.
- **No reformatear, reindentar, reserializar ni "limpiar" el XML después de
  firmar.** Hay que enviar el archivo byte a byte como salió del firmador.

El RPI resuelve esa referencia **contra la parte `xml` del envío**, sin importar
qué nombre de archivo diga la `URI`: lo que se verifica son los bytes recibidos.

## Algoritmos permitidos

| Componente | Algoritmo |
|------------|-----------|
| Canonicalización de `SignedInfo` | `http://www.w3.org/2001/10/xml-exc-c14n#` (Exclusive C14N) |
| Algoritmo de firma | `http://www.w3.org/2001/04/xmldsig-more#rsa-sha256` o superior |
| Digest de las referencias | `http://www.w3.org/2001/04/xmlenc#sha256` o superior |
| Transformación de `SignedProperties` | `http://www.w3.org/2001/10/xml-exc-c14n#` |
| Transformación del documento | **ninguna** (digest sobre los bytes crudos) |

**Política**: solo **SHA-256 o superior** (SHA-384, SHA-512). **SHA-1 se rechaza**,
tanto en el digest de las referencias como en el algoritmo de firma: está roto
para resistencia a colisiones y no sirve para un documento con valor registral.
Un envío firmado con `rsa-sha1` o `#sha1` se rechaza aunque la firma cierre
matemáticamente.

## Certificado del firmante

> ⚠️ **Pendiente de confirmación**. Qué autoridades certificantes acepta el RPI
> está en proceso de definición. Se confirmará antes de homologación.

El certificado del firmante debe cumplir:

- Ser un certificado válido de **firma digital** (no solo certificado SSL).
- Estar emitido por una **autoridad certificante reconocida** por el RPI. La
  lista de raíces confiables es **configuración del servicio**, no está en el
  contrato.
- Estar vigente al momento del envío (no vencido, no revocado).
- Pertenecer al **escribano autorizante** declarado en el XML.

El certificado se incluye en `<ds:KeyInfo><ds:X509Data><ds:X509Certificate>`, y
puede venir acompañado de las intermedias de su cadena (el RPI las usa para
construir la ruta hasta una raíz confiable). **Siempre incluir al menos el
certificado del firmante.**

### El CUIT/CUIL va en el `serialNumber`

El RPI extrae el identificador del firmante del atributo **`serialNumber`** del
subject del certificado, que admite los dos prefijos en uso:

```
serialNumber=CUIL 20403181871
serialNumber=CUIT 30716363380
```

Ese valor se compara con `EscribanoAutorizante/CUIT` del XML, comparando los
**11 dígitos** (el match es agnóstico de guiones y del prefijo). Si el
certificado no expone un `serialNumber` interpretable, el RPI lo registra y
omite esa comparación — no rechaza por eso.

## Estructura de la firma

Ejemplo de archivo de firma detached (`firma_xml_escribano`):

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<ds:Signature xmlns:ds="http://www.w3.org/2000/09/xmldsig#" Id="id-396e95...">
  <ds:SignedInfo>
    <ds:CanonicalizationMethod Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
    <ds:SignatureMethod Algorithm="http://www.w3.org/2001/04/xmldsig-more#rsa-sha256"/>

    <!-- Referencia al documento: SIN Transforms → digest sobre los bytes crudos -->
    <ds:Reference Id="r-id-396e95...-1" URI="minuta_id_691.xml">
      <ds:DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha256"/>
      <ds:DigestValue>BASE64_DEL_SHA256_DE_LOS_BYTES_DEL_XML</ds:DigestValue>
    </ds:Reference>

    <!-- Referencia a las propiedades firmadas XAdES: canonicalizada -->
    <ds:Reference Type="http://uri.etsi.org/01903#SignedProperties"
                  URI="#xades-id-396e95...">
      <ds:Transforms>
        <ds:Transform Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
      </ds:Transforms>
      <ds:DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha256"/>
      <ds:DigestValue>BASE64_DEL_SHA256_DE_SIGNEDPROPERTIES_C14N</ds:DigestValue>
    </ds:Reference>
  </ds:SignedInfo>

  <ds:SignatureValue Id="value-id-396e95...">BASE64_DE_LA_FIRMA_RSA</ds:SignatureValue>

  <ds:KeyInfo>
    <ds:X509Data>
      <ds:X509Certificate>BASE64_DEL_CERT_DEL_ESCRIBANO</ds:X509Certificate>
      <ds:X509Certificate>BASE64_DE_LA_INTERMEDIA</ds:X509Certificate>
      <ds:X509Certificate>BASE64_DE_LA_RAIZ</ds:X509Certificate>
    </ds:X509Data>
  </ds:KeyInfo>

  <ds:Object>
    <xades:QualifyingProperties xmlns:xades="http://uri.etsi.org/01903/v1.3.2#"
                                Target="#id-396e95...">
      <xades:SignedProperties Id="xades-id-396e95...">
        <xades:SignedSignatureProperties>
          <xades:SigningTime>2026-10-08T15:02:21Z</xades:SigningTime>
          <xades:SigningCertificateV2>
            <xades:Cert>
              <xades:CertDigest>
                <ds:DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha512"/>
                <ds:DigestValue>BASE64_DEL_DIGEST_DEL_CERT</ds:DigestValue>
              </xades:CertDigest>
              <xades:IssuerSerialV2>BASE64_DEL_ISSUERSERIAL</xades:IssuerSerialV2>
            </xades:Cert>
          </xades:SigningCertificateV2>
        </xades:SignedSignatureProperties>
        <xades:SignedDataObjectProperties>
          <xades:DataObjectFormat ObjectReference="#r-id-396e95...-1">
            <xades:MimeType>text/xml</xades:MimeType>
          </xades:DataObjectFormat>
        </xades:SignedDataObjectProperties>
      </xades:SignedProperties>
    </xades:QualifyingProperties>
  </ds:Object>
</ds:Signature>
```

## Verificación que hace el RPI

Cuando el RPI recibe el testimonio:

1. Verifica que llegue la parte `firma_xml_escribano`. Si falta, rechaza.
2. Verifica que el XML **no** contenga un `<ds:Signature>` adentro. Si lo
   contiene, rechaza.
3. Verifica que los algoritmos declarados estén permitidos (SHA-256 o superior).
   Si hay SHA-1, rechaza.
4. Calcula el SHA-256 de los **bytes crudos** de la parte `xml` y lo compara con
   el `<ds:DigestValue>` de la referencia al documento.
5. Canonicaliza `<xades:SignedProperties>` con Exclusive C14N, calcula su digest
   y lo compara con el de su referencia.
6. Verifica que el certificado referenciado en `SigningCertificateV2` sea **el
   mismo** que viene en `<ds:KeyInfo>`.
7. Canonicaliza `<ds:SignedInfo>` y verifica la `<ds:SignatureValue>` con la
   clave pública del certificado del firmante.
8. Construye la cadena del certificado hasta una **raíz confiable** de la
   configuración del servicio, y verifica su vigencia.
9. Compara el CUIT/CUIL del `serialNumber` del certificado con
   `EscribanoAutorizante/CUIT` del XML.

Si cualquiera de estos pasos falla, el RPI rechaza el testimonio (ver
[07 — Respuestas y errores](07-respuestas-y-errores.md)).

## Pendientes de definición

Estas dos piezas están previstas pero **todavía no se verifican**:

- **Sello de tiempo (XAdES-T)**. La firma lleva `<xades:SigningTime>`, que es
  una fecha **declarada por el firmante** y por lo tanto no es prueba de cuándo
  se firmó. Un sello de tiempo de una TSA daría esa prueba. Queda pendiente de
  definir qué TSA se acepta.
- **Segunda firma (de aplicación, del Colegio)**. El diseño contempla que el
  envío lleve además una firma del sistema del Colegio, que acredite que el
  testimonio pasó por su plataforma. El contrato reservará otra parte del
  multipart para eso. **No está implementada ni es exigible todavía.**

## Bibliotecas recomendadas

Para implementación del lado del sistema del Colegio:

- **Java**: Apache Santuario (`xml-security`) + DSS para XAdES.
- **C# / .NET**: `System.Security.Cryptography.Xml.SignedXml`.
- **Python**: `signxml`.
- **Node.js**: `xml-crypto` (ver la nota de abajo).
- **PHP**: `php-xmlseclibs`.

> **Nota para quien implemente en Node.js**: `xml-crypto` **no resuelve
> referencias externas**. Su verificación busca la `URI` como ID de un elemento
> del mismo documento, y además canonicaliza el contenido referenciado — lo que
> daría un digest distinto al de una referencia sin transforms. Para detached hay
> que resolver la referencia a mano: calcular el digest sobre los bytes del
> archivo y usar `xml-crypto` solo para la canonicalización de `SignedInfo` y
> `SignedProperties`.

## Errores comunes a evitar

- **Normalizar los saltos de línea**: es el error más común en detached. Si el
  archivo se firmó con CRLF, hay que enviarlo con CRLF. Muchas librerías y
  editores convierten a LF sin avisar.
- **Reserializar el XML después de firmar**: parsear y volver a escribir el
  documento cambia espacios, orden de atributos o comillas, y rompe el digest.
- **No incluir el certificado en `<KeyInfo>`**: sin él el RPI no puede verificar
  la firma.
- **Usar SHA-1**: se rechaza.
- **Mandar la firma dentro del XML**: en v3.1 el modo es detached; una firma
  envuelta se rechaza.
- **Cambiar el nombre del archivo y asumir que importa**: el RPI ignora el nombre
  de la `URI` y verifica contra la parte `xml` recibida. Cambiar el nombre no
  rompe nada; cambiar los bytes sí.

## Nota sobre verificación de certificado

La implementación actual del servicio del RPI verifica la cadena del certificado
hasta una raíz confiable y su vigencia, pero **no verifica**:

- Habilitación de la autoridad certificante por ONTI.
- Estado de revocación (CRL/OCSP).

Esta limitación está documentada como deuda técnica del servicio. Para clientes
implementadores: asumir que en producción el RPI verificará también la
revocación. Asegurar que el certificado del escribano esté vigente al momento de
cada envío (renovarlo antes del vencimiento) y no esté revocado.

Si alguno de estos checks falla, el RPI rechazará el testimonio con un código
específico (`CERTIFICADO_NO_VIGENTE`, `CERTIFICADO_REVOCADO`,
`CERTIFICADO_NO_RECONOCIDO`), ya documentados en
[07 — Respuestas y errores](07-respuestas-y-errores.md).

## Próximos pasos

- Para el adjunto PDF y su verificación, andá a [06 — Adjunto PDF](06-adjunto-pdf.md).
- Para los códigos de error de firma, andá a [07 — Respuestas y errores](07-respuestas-y-errores.md).

---

[← Índice de la documentación](README.md)
