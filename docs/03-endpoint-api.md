# 03 — Endpoint API

## URL del endpoint

> ⚠️ **Pendiente de definición**. Las URLs finales se confirmarán antes de
> la entrega definitiva del contrato.

| Ambiente | URL |
|----------|-----|
| Staging / Homologación | `https://[POR-DEFINIR].jusneuquen.gov.ar/api/v1/testimonios` |
| Producción | `https://[POR-DEFINIR].jusneuquen.gov.ar/api/v1/testimonios` |

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

### Caso de éxito (202 Accepted)

```http
HTTP/1.1 202 Accepted
Content-Type: application/json

{
  "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
  "estado": "recibido",
  "recibidoEn": "2026-06-15T10:23:45Z",
  "mensaje": "Testimonio recibido y en proceso de validación."
}
```

El campo `identificadorEnvio` debe coincidir con el `MetadatosEnvio/IdentificadorEnvio`
que el sistema del Colegio incluyó en el XML.

### Caso de idempotencia (200 OK)

Si el sistema del Colegio reenvía el mismo testimonio (mismo `identificadorEnvio`)
y el original ya fue procesado, el RPI devuelve **el estado actual** del testimonio
original, **sin volver a procesarlo**:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
  "estado": "sincronizado",
  "recibidoEn": "2026-06-15T10:23:45Z",
  "identificadorRPI": "12345",
  "mensaje": "Testimonio ya recibido anteriormente. Devolviendo estado actual."
}
```

### Casos de error

Ver [07 — Respuestas y errores](07-respuestas-y-errores.md) para el catálogo
completo de códigos HTTP y respuestas de error.

## Límites técnicos

| Recurso | Límite |
|---------|--------|
| Tamaño total de la petición | 50 MB (XML + PDF + overhead) |
| Tamaño del XML solo | 5 MB |
| Tamaño del PDF solo | 45 MB |
| Timeout de respuesta | 30 segundos |

Si la petición excede los límites, el RPI responde 413 Payload Too Large.

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
