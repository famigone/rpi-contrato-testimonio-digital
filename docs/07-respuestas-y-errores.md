# 07 — Respuestas y errores

## Dos tipos de error, dos conductas

Antes del catálogo, la distinción que define qué hacer:

- **Rechazo (4xx, salvo 429)**: el envío tiene un problema que el Colegio debe
  corregir. El testimonio **no ingresó** al registro. El rechazo es definitivo para
  ese `IdentificadorEnvio`: **corregir y enviar con un `IdentificadorEnvio` nuevo**.
- **Problema transitorio (5xx, 429, timeout, error de red)**: el envío no tiene nada
  que corregir. **Reintentar el mismo envío con el mismo `IdentificadorEnvio`.**

Los rechazos de este capítulo ocurren **antes** de que el testimonio tenga Entrada
General. Las observaciones del calificador, que ocurren después, no son errores de
la API: llegan por callback (ver [02 — Qué pasa si algo falla](02-flujo-end-to-end.md#qué-pasa-si-algo-falla)
y [08 — Notificaciones](08-notificaciones-callback.md)).

## Códigos HTTP de respuesta

| Código | Significado | Acción del cliente |
|--------|-------------|--------------------|
| 200 OK | Testimonio ya recibido y aceptado previamente (idempotencia) | Usar la respuesta para sincronizar el estado local. No reenviar. |
| 202 Accepted | Testimonio validado y aceptado | Guardar el `identificadorEnvio` y esperar callbacks. |
| 400 Bad Request | XML mal formado o inválido (XSD), multipart inválido, hash o firma del PDF | Corregir y enviar con un `identificadorEnvio` nuevo. |
| 401 Unauthorized | Credenciales faltantes o inválidas, o firma XML inválida | Revisar autenticación y firma. Corregir y enviar con un `identificadorEnvio` nuevo. |
| 403 Forbidden | Cliente autenticado pero sin permiso para esta operación | Contactar al equipo del RPI. No reintentar. |
| 409 Conflict | `identificadorEnvio` ya usado con contenido distinto | Generar un `identificadorEnvio` nuevo y reenviar. |
| 413 Payload Too Large | Petición excede el límite (50 MB total) | Reducir el tamaño del PDF y enviar con un `identificadorEnvio` nuevo. |
| 422 Unprocessable Entity | XML válido pero falla una regla de negocio o la tasa | Revisar el error específico, corregir y enviar con un `identificadorEnvio` nuevo. |
| 429 Too Many Requests | Rate limit del RPI alcanzado | Reintentar con el mismo `identificadorEnvio`, respetando `Retry-After`. |
| 500 Internal Server Error | Error del lado del RPI | Reintentar con el mismo `identificadorEnvio` y backoff exponencial. |
| 502 Bad Gateway | RPI temporalmente no disponible | Ídem. |
| 503 Service Unavailable | RPI en mantenimiento o dependencia no disponible | Ídem, respetando `Retry-After` si viene. |
| 504 Gateway Timeout | Timeout del RPI procesando | Ídem. |

## Formato del cuerpo de error

Todos los errores devuelven un cuerpo JSON con esta estructura:

```json
{
  "error": {
    "codigo": "XML_INVALIDO",
    "mensaje": "El elemento 'Partes' debe tener al menos un hijo 'Parte'.",
    "detalle": {
      "linea": 42,
      "columna": 12,
      "elemento": "/TestimonioDigital/Actos/Acto/Partes"
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
| `IDENTIFICADOR_DUPLICADO_CONTENIDO_DISTINTO` | 409 | UUID ya usado para un envío con contenido distinto. |
| `AUTENTICACION_REQUERIDA` | 401 | Falta el header de autenticación. |
| `AUTENTICACION_INVALIDA` | 401 | El token o certificado de autenticación es inválido. |
| `SIN_PERMISO` | 403 | Cliente autenticado pero sin permiso para enviar testimonios. |
| `LIMITE_TAMANO` | 413 | La petición supera 50 MB. |
| `RATE_LIMIT` | 429 | Demasiadas peticiones del mismo cliente. Respetar `Retry-After`. |

### Errores de negocio (422)

| Código | Descripción |
|--------|-------------|
| `ESCRIBANO_NO_REGISTRADO` | El escribano declarado no figura en el catálogo del RPI. Requiere acción manual. |
| `ROGANTE_NO_ENCONTRADO` | El rogante declarado no está registrado en el RPI (se resuelve por CUIT). |
| `MATRICULA_NO_EXISTENTE` | La matrícula del inmueble no existe en el RPI. |
| `CERTIFICACION_VENCIDA` | Una certificación registral previa está vencida: la de **dominio** (`Acto/CertificacionDominio`) o la de **inhibición** de algún transmitente (`Acto/Partes/Parte/CertificacionInhibicion`). |
| `VERSION_CONTRATO_NO_SOPORTADA` | El RPI no soporta la versión del contrato declarada. |
| `CODIGO_ACTO_NO_HABILITADO` | El `<Codigo>` del acto existe en el catálogo pero **no está habilitado** para testimonio digital. |
| `MONEDA_SIN_COTIZACION` | Si moneda=USD, cotización es obligatoria. |
| `PERSONA_HUMANA_SIN_NOMBRES` | Tipo=H requiere Nombres. |
| `CERT_CATASTRAL_DATOS_FALTANTES` | Si Emitido=true, Numero y CodigoValidacion obligatorios. |
| `CERT_CATASTRAL_OBS_FALTANTE` | Si TieneObservaciones=true, Observaciones obligatorio. |
| `VISADO_SIN_NUMERO_TRAMITE` | Si Tipo=R, NumeroTramite obligatorio. |
| `INMUEBLE_SIN_IDENTIFICACION` | Inmueble debe tener Matricula o (Tomo+Folio+Finca). |
| `PROPORCIONES_NO_SUMAN_UNO` | Suma de proporciones de partes ADQUIRENTE debe ser 1, **por acto**. |
| `NUMERO_ACTO_DUPLICADO` | El atributo `numero` debe ser único entre los actos del testimonio. |
| `ROL_NO_VALIDO_PARA_ACTO` | El `rol` de la parte no corresponde al acto (ej. ACREEDOR en una compraventa, código 1028). |
| `ACTO_SIN_ADQUIRENTE` | El acto exige al menos una parte con rol ADQUIRENTE (ej. compraventa, código 1028). |
| `CUIT_FIRMANTE_NO_COINCIDE` | El CUIT del certificado debe coincidir con el del XML. |
| `TASA_INEXISTENTE` | El número de tasa declarado no existe en el sistema de tasas. |
| `TASA_NO_PAGADA` | La tasa declarada existe pero no está paga. |
| `TASA_YA_UTILIZADA` | La tasa declarada ya fue aplicada a otro trámite. |

Los errores de tasa también pueden llegar después del 202, por el callback
`validacion_fallida`, cuando el sistema de tasas no respondió durante la recepción
(estado `aceptado_tasa_pendiente`).

### Errores del servidor (5xx)

| Código | HTTP | Descripción |
|--------|------|-------------|
| `ERROR_INTERNO` | 500 | Error inesperado del RPI. Reintentar con backoff. |
| `BD_NO_DISPONIBLE` | 503 | Base de datos temporalmente no disponible. |
| `MANTENIMIENTO` | 503 | RPI en mantenimiento programado. Header `Retry-After` indica cuándo reintentar. |
| `TIMEOUT_INTERNO` | 504 | El RPI se quedó procesando demasiado tiempo. |

Un error 5xx nunca es un rechazo del testimonio: indica que el RPI no pudo
completar el procesamiento por una causa propia.

## Política de reintentos

### Rechazos (4xx, salvo 429) — corregir y enviar de nuevo

Un 4xx indica que el envío tiene que **cambiar** antes de reenviarse. Reenviarlo
idéntico devuelve el mismo error.

- El rechazo es definitivo para ese `IdentificadorEnvio`.
- El envío corregido es un envío nuevo y **lleva un `IdentificadorEnvio` nuevo**.
- Si se reusa el identificador con contenido distinto, el RPI responde 409
  (`IDENTIFICADOR_DUPLICADO_CONTENIDO_DISTINTO`).

### Problemas transitorios — reintentar el mismo envío

Errores 5xx, 429, timeouts del cliente y errores de red. Se reintenta **el mismo
envío con el mismo `IdentificadorEnvio`**: si el RPI llegó a procesarlo, devuelve
el resultado original sin duplicarlo.

Para 429 y 503, respetar el header `Retry-After` si viene. En los demás casos,
backoff exponencial:

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

### Resumen

| Situación | ¿Mismo `IdentificadorEnvio`? |
|-----------|------------------------------|
| Timeout, error de red, 5xx, 429 | Sí: es el mismo envío |
| Rechazo 4xx, después de corregir | No: es un envío nuevo |
| Rechazo diferido (callback `validacion_fallida`), después de corregir | No: es un envío nuevo |
| Observación registral (callback `inscripcion_provisoria`) | No aplica: se subsana por mesa de entradas (ver [02](02-flujo-end-to-end.md#observación-registral-después-de-la-eg)) |

## Ejemplos de respuestas de error

### XML inválido

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": {
    "codigo": "XML_INVALIDO",
    "mensaje": "El elemento 'Partes' debe tener al menos un hijo 'Parte'.",
    "detalle": {
      "elemento": "/TestimonioDigital/Actos/Acto/Partes"
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

### Tasa no pagada

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json

{
  "error": {
    "codigo": "TASA_NO_PAGADA",
    "mensaje": "La tasa registral declarada no registra pago.",
    "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2026-06-15T10:23:45Z"
  }
}
```

## Próximos pasos

- Para los callbacks que el RPI envía al Colegio, andá a
  [08 — Notificaciones de callback](08-notificaciones-callback.md).

---

[← Índice de la documentación](README.md)