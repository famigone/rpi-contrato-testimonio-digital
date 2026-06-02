# 07 — Respuestas y errores

## Códigos HTTP de respuesta

| Código | Significado | Acción del cliente |
|--------|-------------|--------------------|
| 200 OK | Testimonio ya recibido previamente (idempotencia) | Usar la respuesta para sincronizar estado local. No reenviar. |
| 202 Accepted | Testimonio recibido y aceptado para procesamiento | Guardar el `identificadorEnvio` y esperar callbacks. |
| 400 Bad Request | XML inválido (XSD), hash de PDF no coincide, JSON mal formado | Corregir y reenviar. No reintentar sin cambios. |
| 401 Unauthorized | Credenciales faltantes, inválidas, o firma XML inválida | Revisar autenticación y firma. No reintentar sin cambios. |
| 403 Forbidden | Cliente autenticado pero sin permiso para esta operación | Contactar al equipo del RPI. No reintentar. |
| 409 Conflict | `identificadorEnvio` ya usado con contenido distinto | Generar nuevo `identificadorEnvio` y reenviar. |
| 413 Payload Too Large | Petición excede el límite (50 MB total) | Reducir tamaño del PDF y reenviar. |
| 422 Unprocessable Entity | El XML es válido sintácticamente pero falla validación de negocio | Revisar el error específico y corregir datos. |
| 429 Too Many Requests | Rate limit del RPI alcanzado | Esperar y reintentar respetando el header `Retry-After`. |
| 500 Internal Server Error | Error del lado del RPI | Reintentar con backoff exponencial. |
| 502 Bad Gateway | RPI temporalmente no disponible | Reintentar con backoff exponencial. |
| 503 Service Unavailable | RPI en mantenimiento | Reintentar con backoff exponencial. |
| 504 Gateway Timeout | Timeout del RPI procesando | Reintentar con backoff exponencial. |

## Formato del cuerpo de error

Todos los errores devuelven un cuerpo JSON con esta estructura:

```json
{
  "error": {
    "codigo": "XML_INVALIDO",
    "mensaje": "El elemento 'Adquirentes' debe tener al menos un hijo.",
    "detalle": {
      "linea": 42,
      "columna": 12,
      "elemento": "/TestimonioDigital/Acto/Compraventa/Adquirentes"
    },
    "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2026-06-15T10:23:45Z"
  }
}
```

El campo `detalle` es opcional y varía según el código de error.

## Catálogo de códigos de error

### Errores de petición (4xx)

| Código | HTTP | Descripción |
|--------|------|-------------|
| `XML_INVALIDO` | 400 | El XML no cumple el XSD. `detalle` indica línea/columna/elemento. |
| `XML_NO_PARSEABLE` | 400 | El XML está mal formado (no es XML válido). |
| `FIRMA_INVALIDA` | 401 | La firma XML-DSig no se puede verificar. `detalle` indica causa. |
| `FIRMA_FALTANTE` | 401 | No se encontró bloque `Signature` en el XML. |
| `CERTIFICADO_NO_VIGENTE` | 401 | El certificado del firmante está vencido. |
| `CERTIFICADO_REVOCADO` | 401 | El certificado del firmante está revocado. |
| `CERTIFICADO_NO_RECONOCIDO` | 401 | La autoridad certificante no es reconocida. |
| `CUIT_NO_COINCIDE` | 401 | El CUIT del certificado no coincide con el del XML. |
| `HASH_PDF_NO_COINCIDE` | 400 | El hash declarado en el XML no coincide con el PDF recibido. |
| `PDF_NO_FIRMADO` | 400 | El PDF no tiene firma digital. |
| `PDF_FIRMA_INVALIDA` | 400 | El PDF tiene firma pero es inválida. |
| `MULTIPART_INVALIDO` | 400 | La petición multipart está mal formada o falta una parte. |
| `IDENTIFICADOR_INVALIDO` | 400 | El `IdentificadorEnvio` no es un UUID v4 válido. |
| `IDENTIFICADOR_DUPLICADO_CONTENIDO_DISTINTO` | 409 | UUID ya usado para un testimonio con contenido distinto. |
| `AUTENTICACION_REQUERIDA` | 401 | Falta el header de autenticación. |
| `AUTENTICACION_INVALIDA` | 401 | El token o certificado de autenticación es inválido. |
| `SIN_PERMISO` | 403 | Cliente autenticado pero sin permiso para enviar testimonios. |
| `LIMITE_TAMANO` | 413 | La petición supera 50 MB. |
| `RATE_LIMIT` | 429 | Demasiadas peticiones del mismo cliente. Respetar `Retry-After`. |

### Errores de negocio (422)

| Código | Descripción |
|--------|-------------|
| `ESCRIBANO_NO_REGISTRADO` | El escribano declarado no figura en el catálogo del RPI. Requiere acción manual. |
| `MATRICULA_NO_EXISTENTE` | La matrícula del inmueble no existe en el RPI. |
| `CERTIFICACION_VENCIDA` | La certificación registral previa está vencida. |
| `VERSION_CONTRATO_NO_SOPORTADA` | El RPI no soporta la versión del contrato declarada. |

### Errores del servidor (5xx)

| Código | HTTP | Descripción |
|--------|------|-------------|
| `ERROR_INTERNO` | 500 | Error inesperado del RPI. Reintentar con backoff. |
| `BD_NO_DISPONIBLE` | 503 | Base de datos temporalmente no disponible. |
| `MANTENIMIENTO` | 503 | RPI en mantenimiento programado. Header `Retry-After` indica cuándo reintentar. |
| `TIMEOUT_INTERNO` | 504 | El RPI se quedó procesando demasiado tiempo. |

## Política de reintentos

### Errores 4xx — NO reintentar

Los errores 4xx (excepto 429) indican que el cliente tiene que **cambiar algo**
antes de reenviar. Reintentar idéntico no va a cambiar el resultado.

**Excepción**: HTTP 429 (rate limit) sí se reintenta, respetando el header
`Retry-After`.

### Errores 5xx — Reintentar con backoff exponencial

Los errores 5xx indican problemas transitorios del lado del RPI. Reintentar
con backoff exponencial:

| Intento | Espera antes del próximo |
|---------|--------------------------|
| 1 | 30 segundos |
| 2 | 1 minuto |
| 3 | 2 minutos |
| 4 | 5 minutos |
| 5 | 15 minutos |
| 6 | 30 minutos |
| 7+ | 1 hora (capear ahí) |

Después de varios reintentos fallidos (por ejemplo 10), conviene alertar a un
operador del sistema del Colegio para que investigue.

### Idempotencia siempre

Los reintentos deben usar el **mismo `IdentificadorEnvio`** que el envío
original. El RPI detecta el reenvío y devuelve el resultado del envío original
sin reprocesar.

Si por error se reenvía con el mismo `IdentificadorEnvio` **pero contenido
distinto** (por ejemplo, otro PDF), el RPI rechaza con HTTP 409
(`IDENTIFICADOR_DUPLICADO_CONTENIDO_DISTINTO`). En ese caso hay que generar un
nuevo UUID.

## Ejemplos de respuestas de error

### XML inválido

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": {
    "codigo": "XML_INVALIDO",
    "mensaje": "El elemento 'CertificacionRegistralPrevia' es obligatorio pero no se encontró.",
    "detalle": {
      "elemento": "/TestimonioDigital/CertificacionRegistralPrevia"
    },
    "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2026-06-15T10:23:45Z"
  }
}
```

### Firma inválida

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "error": {
    "codigo": "FIRMA_INVALIDA",
    "mensaje": "No se pudo verificar la firma XML-DSig.",
    "detalle": {
      "causa": "El hash del documento canonicalizado no coincide con el DigestValue."
    },
    "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2026-06-15T10:23:45Z"
  }
}
```

### Hash de PDF no coincide

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": {
    "codigo": "HASH_PDF_NO_COINCIDE",
    "mensaje": "El hash SHA-256 del PDF recibido no coincide con el declarado en el XML.",
    "detalle": {
      "hashDeclarado": "3a7bd3e2360a3d29eea436fcfb7e44c735d117c42d1c1835420b6b9942dd4f1b",
      "hashCalculado": "8e34a7c1f9b8e0c8de72e80e2b9bf28c1c34e2f0b7d5a3c1f8e9b8e0c8de72e80"
    },
    "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2026-06-15T10:23:45Z"
  }
}
```

## Próximos pasos

- Para los callbacks que el RPI envía al Colegio, andá a
  [08 — Notificaciones de callback](08-notificaciones-callback.md).
