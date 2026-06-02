# 08 — Notificaciones de callback

## Qué son

Después de que el RPI acepta un testimonio (HTTP 202), el procesamiento del
trámite registral sigue de manera asincrónica. Cuando hay eventos relevantes
en el ciclo de vida del testimonio (inscripción provisoria, definitiva,
rechazo registral, etc.), el RPI **notifica al sistema del Colegio** vía un
callback HTTP.

El sistema del Colegio debe **exponer un endpoint** que el RPI invoca para
entregar estos eventos.

## Endpoint del callback (lado del Colegio)

El sistema del Colegio debe proveer al RPI:

- Una **URL** donde recibir las notificaciones.
- Un **mecanismo de autenticación** (idealmente el mismo que se usa en la
  dirección Colegio → RPI, pero invertido).

> ⚠️ **Pendiente de definición**. La URL final del callback del Colegio y el
> mecanismo de autenticación se definirán de común acuerdo antes de la entrega
> definitiva.

## Tipos de eventos

| Tipo de evento | Cuándo se dispara |
|----------------|-------------------|
| `validacion_completada` | El RPI completó validaciones (XSD, firma, hash) y el testimonio entró a la cola registral. |
| `sincronizacion_completada` | El testimonio fue sincronizado con el sistema registral interno del RPI. |
| `entrada_general_asignada` | El RPI asignó número de Entrada General al trámite. |
| `inscripcion_provisoria` | Hay observaciones registrales pendientes de subsanar (VIP/VIO). |
| `inscripcion_definitiva` | El trámite quedó inscripto definitivamente. |
| `rechazo_registral` | El trámite fue rechazado por el calificador (causal grave). |
| `validacion_fallida` | Las validaciones del RPI fallaron de manera persistente y el testimonio no podrá procesarse. |

## Estructura del payload

Todos los callbacks tienen esta estructura básica:

```json
{
  "evento": "inscripcion_definitiva",
  "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-06-16T14:30:22Z",
  "datos": {
    // contenido específico del evento
  }
}
```

### Ejemplo: `validacion_completada`

```json
{
  "evento": "validacion_completada",
  "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-06-15T10:24:01Z",
  "datos": {
    "estado": "validado"
  }
}
```

### Ejemplo: `entrada_general_asignada`

```json
{
  "evento": "entrada_general_asignada",
  "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-06-15T10:30:15Z",
  "datos": {
    "entradaGeneral": {
      "numero": 12345,
      "anio": 2026,
      "presentacion": "2026-06-15"
    }
  }
}
```

### Ejemplo: `inscripcion_definitiva`

```json
{
  "evento": "inscripcion_definitiva",
  "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-06-16T14:30:22Z",
  "datos": {
    "entradaGeneral": {
      "numero": 12345,
      "anio": 2026,
      "presentacion": "2026-06-15"
    },
    "fechaInscripcion": "2026-06-16",
    "matriculas": [
      {
        "matricula": "12-3456",
        "departamento": "Confluencia",
        "asientoNumero": 7
      }
    ]
  }
}
```

### Ejemplo: `inscripcion_provisoria`

```json
{
  "evento": "inscripcion_provisoria",
  "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-06-16T11:15:00Z",
  "datos": {
    "entradaGeneral": {
      "numero": 12345,
      "anio": 2026,
      "presentacion": "2026-06-15"
    },
    "fechaProvisoria": "2026-06-16",
    "fechaVencimientoVIP": "2026-08-15",
    "observaciones": [
      {
        "codigo": "FALTA_TASA",
        "descripcion": "Falta acreditar pago de tasa registral con número de tasa indicado."
      },
      {
        "codigo": "CLAUSULA_AMBIGUA",
        "descripcion": "Cláusula tercera del testimonio requiere aclaración sobre la proporción de adquisición."
      }
    ]
  }
}
```

### Ejemplo: `rechazo_registral`

```json
{
  "evento": "rechazo_registral",
  "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-06-16T15:00:00Z",
  "datos": {
    "entradaGeneral": {
      "numero": 12345,
      "anio": 2026,
      "presentacion": "2026-06-15"
    },
    "fechaRechazo": "2026-06-16",
    "motivo": "Matrícula informada no corresponde al transmitente declarado."
  }
}
```

### Ejemplo: `validacion_fallida`

```json
{
  "evento": "validacion_fallida",
  "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-06-15T10:25:00Z",
  "datos": {
    "motivo": "Escribano no registrado en catálogo del RPI.",
    "codigo": "ESCRIBANO_NO_REGISTRADO"
  }
}
```

## Headers del callback

El RPI envía las notificaciones con estos headers:

```
POST /callback-rpi HTTP/1.1
Host: [URL_DEL_COLEGIO]
Content-Type: application/json
Authorization: [mecanismo a definir]
X-RPI-Evento: inscripcion_definitiva
X-RPI-IdentificadorEnvio: 550e8400-e29b-41d4-a716-446655440000
X-RPI-Timestamp: 2026-06-16T14:30:22Z
User-Agent: RPI-Neuquen/1.0
```

Los headers `X-RPI-*` son redundantes con el cuerpo JSON pero útiles para
ruteo o logging del lado del Colegio sin tener que parsear el body.

## Respuesta esperada del Colegio

El endpoint del Colegio debe responder rápido (idealmente < 5 segundos) con
uno de estos códigos:

| Código | Significado |
|--------|-------------|
| 200 OK | Notificación recibida y procesada. |
| 202 Accepted | Notificación recibida, se procesará asincrónicamente. |
| 4xx | Error en la notificación (no se reintenta). |
| 5xx | Error transitorio del Colegio (se reintenta con backoff). |

## Política de reintentos del lado del RPI

Si el endpoint del Colegio no responde (timeout) o responde con 5xx, el RPI
reintenta con backoff exponencial:

| Intento | Espera |
|---------|--------|
| 1 | inmediato |
| 2 | 1 minuto |
| 3 | 5 minutos |
| 4 | 15 minutos |
| 5 | 1 hora |
| 6 | 6 horas |
| 7 | 24 horas |

Después del intento 7, la notificación queda en estado **agotada** y requiere
intervención manual.

## Idempotencia del lado del Colegio

El RPI puede reenviar la misma notificación (por ejemplo, si no recibió la
respuesta del Colegio a tiempo). El sistema del Colegio debe **detectar
duplicados** por el par `(identificadorEnvio, evento)`.

Si ya procesaste una notificación con esa combinación, responder 200 OK sin
volver a procesar.

## Orden de los eventos

Los eventos llegan en orden cronológico, pero por errores de red o reintentos
pueden llegar **desordenados** o **duplicados**. El sistema del Colegio debe
ser tolerante a esto.

Para garantizar consistencia, el campo `timestamp` indica cuándo se generó el
evento del lado del RPI. El Colegio puede usarlo para detectar eventos viejos
que llegan después de uno más nuevo.

Ejemplo: si ya recibiste `inscripcion_definitiva`, ignorar callbacks
`inscripcion_provisoria` con timestamp anterior (es un reintento tardío).

## Seguridad

> ⚠️ **Pendiente de definición**. Mecanismos posibles:
>
> - **Bearer token** que el RPI envía en el header `Authorization`.
> - **mTLS** con certificado cliente del RPI.
> - **HMAC** del cuerpo con un secret compartido, en header `X-RPI-Signature`.
>
> Se definirá en común con el equipo del Colegio.

El Colegio **debe verificar la autenticación** de los callbacks antes de
procesarlos para evitar inyección de eventos falsos.

## Próximos pasos

- Para términos del dominio que pueden no resultar familiares, andá a
  [09 — Glosario](09-glosario.md).
