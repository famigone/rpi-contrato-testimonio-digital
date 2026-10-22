# 03 — Endpoint API

## URL del endpoint

| Ambiente | URL |
|----------|-----|
| Staging / Homologación | `https://test.jusneuquen.gov.ar/rpi-td/api/v1/testimonios` |
| Producción | `https://[HOST-PRODUCCION-POR-DEFINIR]/rpi-td/api/v1/testimonios` |

> ⚠️ **El host de producción está pendiente de definición.** El de staging ya es
> definitivo.

### Sobre el prefijo `/rpi-td`

El `/rpi-td` **es parte de la URL pública, no del servicio**. Lo agrega el
enrutamiento del RPI: el proxy inverso de infraestructura lo recorta y reenvía
la petición al servicio, que recibe `/api/v1/testimonios`.

Qué implica para quien integra:

- **Usar siempre la URL pública completa**, con el prefijo. Pegarle directo a
  `/api/v1/testimonios` sin el prefijo no funciona desde afuera.
- **No deducir otras rutas sacando el prefijo.** Cualquier endpoint que el RPI
  publique en el futuro va a estar documentado acá con su URL pública completa.
- El prefijo es un detalle del enrutamiento del RPI y **puede cambiar** sin que
  cambie el contrato del testimonio. Si cambia, se avisa por el canal de
  gobernanza y se publica una versión nueva de este capítulo. Conviene que la
  URL base sea **configurable** en el sistema del Colegio y no esté escrita en
  el código.

> **Nota sobre el `v1` del path**: el `/api/v1/` de la URL es la versión de la
> **API HTTP**, independiente de la versión del **contrato** del testimonio
> (**v3**, con namespace `https://contrato.rpi.jusneuquen.gov.ar/testimonio-digital/v3`).
> Son dos ejes distintos: la API puede seguir en `v1` mientras el cuerpo del XML
> usa el contrato v3.

## Método HTTP

```
POST /rpi-td/api/v1/testimonios
```

> El `v1` del path es la versión de la **API HTTP**; el `/rpi-td` es el prefijo
> del enrutamiento público (ver arriba). Ninguno de los dos tiene relación con
> la versión del **contrato** del XML.

## Autenticación

> ⚠️ **Pendiente de definición**. Las dos opciones más probables son:
>
> - **Bearer token** estático provisto por el RPI al Colegio, enviado en el
>   header `Authorization: Bearer <token>`.
> - **mTLS** con certificado cliente del Colegio.
>
> Se definirá según los estándares de seguridad del Poder Judicial y se
> documentará en una versión posterior de este documento.

Mientras se cierra la decisión, asumir que la petición lleva un header de
autenticación que el RPI valida antes de procesar el contenido.

## Formato de la petición

### Content-Type

```
Content-Type: multipart/form-data; boundary=...
```

### Partes del multipart

La petición debe contener exactamente **tres partes**:

> ⚠️ **Los nombres de campo son exactos y en minúsculas**: `xml`, `pdf` y
> `firma_xml_escribano`. El RPI los compara carácter por carácter y **distingue
> mayúsculas de minúsculas**: una parte con `name="XML"` o
> `name="Firma_XML_Escribano"` **no** se toma como la parte equivalente. Las
> partes con un nombre que no sea uno de esos tres se descartan en silencio, y
> el envío se rechaza con `MULTIPART_INVALIDO` (400) por **parte faltante**: el
> rechazo nombra la parte que falta, no el nombre mal escrito que se envió.
> El `filename` de cada parte, en cambio, el RPI no lo lee nunca.

#### Parte 1 — `xml`

- **Nombre del campo**: `xml`
- **Content-Type**: `application/xml`
- **Filename (opcional)**: `testimonio.xml`
- **Contenido**: el XML del testimonio digital, **sin firma adentro**. Desde la
  **v3.1** la firma no va envuelta en el XML: va en la parte 3.

#### Parte 2 — `pdf`

- **Nombre del campo**: `pdf`
- **Content-Type**: `application/pdf`
- **Filename (opcional)**: `testimonio.pdf`
- **Contenido**: el PDF firmado del testimonio.

#### Parte 3 — `firma_xml_escribano`

- **Nombre del campo**: `firma_xml_escribano`
- **Content-Type**: `application/xml`
- **Filename (opcional)**: `testimonio.sig.xml`
- **Contenido**: la firma **XAdES-BES detached** del escribano sobre el XML de
  la parte 1 — un documento cuyo elemento raíz es `<ds:Signature>`.
- **Obligatoria**. Si falta, el RPI rechaza el envío.

Desde la **v3.1** la firma no va dentro del XML sino en esta parte. La
referencia de la firma se resuelve **contra la parte `xml`** del mismo envío,
sin importar qué nombre de archivo declare su atributo `URI`.

> ⚠️ **El XML de la parte 1 se verifica byte a byte.** El digest de la firma se
> calcula sobre los **bytes crudos** del archivo, sin canonicalizar, así que los
> finales de línea (CRLF vs LF), el BOM y la indentación son parte de lo
> firmado. Hay que enviar el XML **exactamente** como salió del firmador: no
> reformatearlo, no reserializarlo, no convertir saltos de línea.
> Ver [05 — Firma digital](05-firma-digital.md).

> ⚠️ **No se acepta la firma envuelta.** Si el XML de la parte 1 contiene un
> `<ds:Signature>` adentro, el RPI rechaza el envío.

### Ejemplo de petición (raw HTTP)

```http
POST /rpi-td/api/v1/testimonios HTTP/1.1
Host: test.jusneuquen.gov.ar
Authorization: Bearer eyJhbGciOi...
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Length: 1234567

------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="xml"; filename="testimonio.xml"
Content-Type: application/xml

<?xml version="1.0" encoding="UTF-8"?>
<TestimonioDigital ...>
  ...
</TestimonioDigital>
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="pdf"; filename="testimonio.pdf"
Content-Type: application/pdf

%PDF-1.7
...
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="firma_xml_escribano"; filename="testimonio.sig.xml"
Content-Type: application/xml

<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<ds:Signature xmlns:ds="http://www.w3.org/2000/09/xmldsig#" Id="id-396e95...">
  ...
</ds:Signature>
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

### Ejemplo con `curl`

```bash
curl -X POST https://test.jusneuquen.gov.ar/rpi-td/api/v1/testimonios \
  -H "Authorization: Bearer $TOKEN" \
  -F "xml=@testimonio.xml;type=application/xml" \
  -F "pdf=@testimonio.pdf;type=application/pdf" \
  -F "firma_xml_escribano=@testimonio.sig.xml;type=application/xml"
```

> `curl -F "campo=@archivo"` manda el archivo **sin modificarlo**, que es lo que
> hace falta para que el digest del XML cierre. Cuidado con los pipes y las
> redirecciones del shell, que sí pueden tocar los finales de línea.

## Respuesta

El RPI completa **todas las validaciones antes de responder** (contrato, firmas,
hash y reglas del acto; ver [02 — Paso 4](02-flujo-end-to-end.md#4-el-rpi-valida-persiste-y-responde)).
Una respuesta 202 significa que el testimonio **fue validado y aceptado**.

### Caso de éxito (202 Accepted)

```http
HTTP/1.1 202 Accepted
Content-Type: application/json

{
  "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
  "estado": "aceptado",
  "recibidoEn": "2026-06-15T10:23:45Z",
  "mensaje": "Testimonio validado y aceptado. La Entrada General se notificará por callback.",
  "tasa": {
    "numero": 15851,
    "estado": "sin_verificar"
  }
}
```

El campo `identificadorEnvio` coincide con el `MetadatosEnvio/IdentificadorEnvio`
que el sistema del Colegio incluyó en el XML.

### La tasa en la respuesta

`tasa` informa el `NumeroTasa` recibido y su estado **en el momento de
responder**. La verificación contra el Sistema Único de Recaudación es
posterior al 202, así que ese estado es:

| `tasa.estado` | Cuándo |
|---|---|
| `sin_codigo` | El XML no trajo `NumeroTasa`. |
| `sin_verificar` | Trajo número; todavía no se consultó al sistema de tasas. |

**Ningún estado de la tasa cambia el código de respuesta ni impide el ingreso.**
Si la tasa resulta impaga, inexistente o ya usada, el testimonio ingresa igual y
la inscripción sale provisional: la ley impide rechazar por cuestiones
tributarias.

El estado verificado **no se informa por esta vía**. El escribano lo conoce por
el resultado de la calificación (una provisional por tasa impaga). Más adelante
se puede agregar una consulta para verlo antes.

### Caso de idempotencia (200 OK)

Si el sistema del Colegio reenvía el mismo testimonio (mismo
`identificadorEnvio`, mismo contenido) y el original fue aceptado, el RPI
devuelve **el estado actual**, **sin volver a procesarlo**:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
  "estado": "ingresado",
  "recibidoEn": "2026-06-15T10:23:45Z",
  "entradaGeneral": {
    "numero": 12345,
    "anio": 2026,
    "numeroPresentacion": 1,
    "fechaPresentacion": "2026-06-15"
  },
  "mensaje": "Testimonio ya recibido anteriormente. Devolviendo estado actual."
}
```

`entradaGeneral` aparece solo cuando ya fue asignada.

Si el original **fue rechazado**, el reenvío idéntico devuelve **el mismo error**
que la primera vez. Para corregir, hay que hacer un envío nuevo con un
`identificadorEnvio` nuevo (ver [02 — Idempotencia](02-flujo-end-to-end.md#idempotencia)).

### Estados del testimonio

| Estado | Significado |
|--------|-------------|
| `aceptado` | Validado y aceptado; pendiente de ingreso al registro. |
| `ingresado` | El registro asignó Entrada General. |
| `inscripto_provisorio` | Calificado con observaciones a subsanar. |
| `inscripto_definitivo` | Inscripto. |
| `rechazado_registral` | Rechazado por el calificador. |

> Antes había dos estados más, `aceptado_tasa_pendiente` y `rechazado`, para el
> caso de una tasa verificada como impaga después del 202. **Ya no existen:** la
> tasa no rechaza, así que no hay rechazos posteriores al 202 y todo rechazo
> estructural es síncrono.

### Casos de error

Ver [07 — Respuestas y errores](07-respuestas-y-errores.md) para el catálogo
completo de códigos HTTP y respuestas de error.

## Límites técnicos

| Recurso | Límite |
|---------|--------|
| Tamaño total de la petición | 50 MB (XML + PDF + firma + overhead) |
| Tamaño del XML solo | 5 MB |
| Tamaño del PDF solo | 45 MB |
| Tamaño de `firma_xml_escribano` solo | 1 MB |
| Tiempo de respuesta del RPI | Típico: menos de 5 segundos. El RPI responde siempre antes de 30 segundos |
| Timeout recomendado para el cliente | 40 segundos |

Si la petición excede los límites de tamaño, el RPI responde 413 Payload Too Large.

Si el cliente corta por timeout, no sabe si el RPI procesó el envío: debe
reintentar con el **mismo** `identificadorEnvio`. Si ya había sido procesado, el
RPI devuelve el resultado original.

## Headers recomendados

| Header | Valor | Comentario |
|--------|-------|------------|
| `Authorization` | `Bearer <token>` | Obligatorio (cuando se confirme método) |
| `Content-Type` | `multipart/form-data; boundary=...` | Obligatorio |
| `Accept` | `application/json` | Recomendado |
| `User-Agent` | `ColegioEscribanosNeuquen/<version>` | Recomendado para trazabilidad |

## Próximos pasos

- Para el formato del XML, andá a [04 — Formato XML](04-formato-xml.md).
- Para los códigos de error, andá a [07 — Respuestas y errores](07-respuestas-y-errores.md).

---

[← Índice de la documentación](README.md)
