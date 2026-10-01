# 03 — Endpoint API

## URL del endpoint

> ⚠️ **Pendiente de definición**. Las URLs finales se confirmarán antes de
> la entrega definitiva del contrato.

| Ambiente | URL |
|----------|-----|
| Staging / Homologación | `https://[POR-DEFINIR].jusneuquen.gov.ar/api/v1/testimonios` |
| Producción | `https://[POR-DEFINIR].jusneuquen.gov.ar/api/v1/testimonios` |

> **Nota sobre el `v1` del path**: el `/api/v1/` de la URL es la versión de la
> **API HTTP**, independiente de la versión del **contrato** del testimonio
> (**v3**, con namespace `https://contrato.rpi.jusneuquen.gov.ar/testimonio-digital/v3`).
> Son dos ejes distintos: la API puede seguir en `v1` mientras el cuerpo del XML
> usa el contrato v3.

## Método HTTP

```
POST /api/v1/testimonios
```

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

La petición debe contener exactamente **dos partes**:

#### Parte 1 — `xml`

- **Nombre del campo**: `xml`
- **Content-Type**: `application/xml`
- **Filename (opcional)**: `testimonio.xml`
- **Contenido**: el XML del testimonio digital, firmado con XML-DSig.

#### Parte 2 — `pdf`

- **Nombre del campo**: `pdf`
- **Content-Type**: `application/pdf`
- **Filename (opcional)**: `testimonio.pdf`
- **Contenido**: el PDF firmado del testimonio.

### Ejemplo de petición (raw HTTP)

```http
POST /api/v1/testimonios HTTP/1.1
Host: [POR-DEFINIR].jusneuquen.gov.ar
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
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

### Ejemplo con `curl`

```bash
curl -X POST https://[POR-DEFINIR]/api/v1/testimonios \
  -H "Authorization: Bearer $TOKEN" \
  -F "xml=@testimonio.xml;type=application/xml" \
  -F "pdf=@testimonio.pdf;type=application/pdf"
```

## Respuesta

El RPI completa **todas las validaciones antes de responder** (contrato, firmas,
hash, reglas del acto y tasa; ver [02 — Paso 4](02-flujo-end-to-end.md#4-el-rpi-valida-persiste-y-responde)).
Una respuesta 202 significa que el testimonio **fue validado y aceptado**.

### Caso de éxito (202 Accepted)

```http
HTTP/1.1 202 Accepted
Content-Type: application/json

{
  "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
  "estado": "aceptado",
  "recibidoEn": "2026-06-15T10:23:45Z",
  "mensaje": "Testimonio validado y aceptado. La Entrada General se notificará por callback."
}
```

El campo `identificadorEnvio` coincide con el `MetadatosEnvio/IdentificadorEnvio`
que el sistema del Colegio incluyó en el XML.

Si el sistema de tasas no respondió a tiempo, el estado es
`aceptado_tasa_pendiente`: el testimonio pasó todas las demás validaciones y la
tasa se verificará después. Si resulta impaga, el RPI lo notifica con el callback
`validacion_fallida`.

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
| `aceptado_tasa_pendiente` | Validado salvo la tasa, que se verificará después. |
| `ingresado` | El registro asignó Entrada General. |
| `inscripto_provisorio` | Calificado con observaciones a subsanar. |
| `inscripto_definitivo` | Inscripto. |
| `rechazado_registral` | Rechazado por el calificador. |
| `rechazado` | Rechazo estructural detectado después de aceptar (tasa impaga). |

### Casos de error

Ver [07 — Respuestas y errores](07-respuestas-y-errores.md) para el catálogo
completo de códigos HTTP y respuestas de error.

## Límites técnicos

| Recurso | Límite |
|---------|--------|
| Tamaño total de la petición | 50 MB (XML + PDF + overhead) |
| Tamaño del XML solo | 5 MB |
| Tamaño del PDF solo | 45 MB |
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
