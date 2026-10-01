# 08 — Notificaciones de callback

## Qué son

Después de que el RPI acepta un testimonio (HTTP 202), el procesamiento registral
sigue de manera asincrónica. Cuando hay eventos relevantes en el ciclo de vida del
testimonio (ingreso al registro, inscripción, observaciones, rechazo), el RPI
**notifica al sistema del Colegio** vía un callback HTTP.

El sistema del Colegio debe **exponer un endpoint** que el RPI invoca para entregar
estos eventos.

## Endpoint del callback (lado del Colegio)

El sistema del Colegio debe proveer al RPI:

- Una **URL** donde recibir las notificaciones.
- Un **mecanismo de autenticación** (idealmente el mismo que se usa en la
  dirección Colegio → RPI, pero invertido).

> ⚠️ **Pendiente de definición**. La URL final del callback del Colegio y el
> mecanismo de autenticación se definirán de común acuerdo antes de la entrega
> definitiva.

## Tipos de eventos

| Tipo de evento | Cuándo se dispara | Trae Entrada General |
|----------------|-------------------|----------------------|
| `entrada_general_asignada` | El registro asignó la Entrada General: el testimonio ingresó con su prioridad. | Sí |
| `inscripcion_provisoria` | El calificador observó el documento: hay observaciones a subsanar dentro de un plazo. | Sí |
| `inscripcion_definitiva` | El trámite quedó inscripto definitivamente. | Sí |
| `rechazo_registral` | El calificador rechazó el trámite (causal grave). | Sí |
| `validacion_fallida` | Rechazo estructural detectado **después** del 202. Hoy, solo la tasa verificada como impaga cuando el testimonio se había aceptado con `aceptado_tasa_pendiente`. | No |

Las validaciones que el RPI hace durante la recepción no generan callbacks: su
resultado viaja en la respuesta HTTP (202 o error 4xx).

`validacion_fallida` es un **rechazo definitivo**: el testimonio no ingresó al
registro y no tiene Entrada General. Para corregir, el Colegio hace un envío nuevo
con un `identificadorEnvio` nuevo. Las fallas internas del RPI **nunca** se
notifican como `validacion_fallida`: el RPI las reintenta por su cuenta.

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

### La Entrada General

Los eventos registrales identifican la Entrada General y la **presentación** a la
que se refieren. Un mismo trámite puede tener varias presentaciones sobre la misma
Entrada General: la primera es el ingreso; las siguientes, subsanaciones de
observaciones.

```json
"entradaGeneral": {
  "numero": 12345,
  "anio": 2026,
  "numeroPresentacion": 1,
  "fechaPresentacion": "2026-06-15"
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
      "numeroPresentacion": 1,
      "fechaPresentacion": "2026-06-15"
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
      "numeroPresentacion": 1,
      "fechaPresentacion": "2026-06-15"
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
      "numeroPresentacion": 1,
      "fechaPresentacion": "2026-06-15"
    },
    "fechaProvisoria": "2026-06-16",
    "fechaVencimientoVIP": "2026-08-15",
    "observaciones": [
      {
        "codigo": "ASENTIMIENTO_FALTANTE",
        "descripcion": "Falta el asentimiento conyugal del transmitente."
      },
      {
        "codigo": "CLAUSULA_AMBIGUA",
        "descripcion": "Cláusula tercera del testimonio requiere aclaración sobre la proporción de adquisición."
      }
    ]
  }
}
```

**Cómo se subsana.** El testimonio conserva su Entrada General y su prioridad
mientras se subsana antes de `fechaVencimientoVIP`. **Por ahora, la subsanación se
presenta por mesa de entradas, citando la Entrada General.** El registro la
incorpora como una nueva presentación sobre la misma Entrada General. La
subsanación por testimonio digital está prevista para una versión futura del
contrato.

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
      "numeroPresentacion": 1,
      "fechaPresentacion": "2026-06-15"
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
  "timestamp": "2026-06-15T10:40:00Z",
  "datos": {
    "codigo": "TASA_NO_PAGADA",
    "motivo": "La tasa registral declarada no registra pago."
  }
}
```

Los códigos de `validacion_fallida` son los del catálogo de errores
([07](07-respuestas-y-errores.md#catálogo-de-códigos-de-error)).

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

Los headers `X-RPI-*` son redundantes con el cuerpo JSON pero útiles para ruteo o
logging del lado del Colegio sin tener que parsear el body.

## Respuesta esperada del Colegio

El endpoint del Colegio debe responder rápido (idealmente < 5 segundos) con uno de
estos códigos:

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
duplicados** por la combinación `(identificadorEnvio, evento, numeroPresentacion)`;
para `validacion_fallida`, que no tiene Entrada General, por
`(identificadorEnvio, evento)`.

Si ya procesaste una notificación con esa combinación, responder 200 OK sin volver
a procesar.

## Orden de los eventos

Los eventos llegan en orden cronológico, pero por errores de red o reintentos
pueden llegar **desordenados** o **duplicados**. El sistema del Colegio debe ser
tolerante a esto.

El campo `timestamp` indica cuándo se generó el evento del lado del RPI. El Colegio
puede usarlo para detectar eventos viejos que llegan después de uno más nuevo.

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

---

[← Índice de la documentación](README.md)