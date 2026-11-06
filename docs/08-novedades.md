# 08 — Novedades del trámite registral

## Qué son

Después de que el RPI acepta un testimonio (HTTP 202), el procesamiento registral
sigue de manera asincrónica y puede durar días o semanas. Cuando hay novedades
—el registro asignó la Entrada General, el trámite salió inscripto, salió
observado— el RPI las deja en una **lista de novedades** que el sistema del
Colegio consulta.

El Colegio **no necesita exponer ningún endpoint**: consulta cuando quiere y
acusa lo que recibió.

> ### ⚠️ Cambio respecto de la versión anterior de este capítulo
>
> La versión anterior describía **callbacks**: el RPI hacía `POST` a una URL del
> Colegio. Ese diseño se reemplazó por el de este capítulo. Si el Colegio ya
> avanzó con la implementación del endpoint receptor, lo que se conserva es el
> **vocabulario de eventos** y la **forma de los datos**; lo que cambia es quién
> inicia la conexión.
>
> **Por qué se cambió**, en una línea: con un callback, el momento en que una
> notificación "tiene efecto" depende de que la infraestructura del Colegio
> estuviera disponible, y eso es exactamente lo que no conviene discutir cuando
> está corriendo el plazo de una inscripción provisional. Con una consulta, el
> hecho verificable es la descarga, que ocurre en el servidor del RPI y queda
> registrada.
>
> Lo que se gana del lado del Colegio: no hay que publicar un endpoint hacia
> afuera, no hay que implementar verificación de firma de los avisos, y una
> caída del sistema del Colegio no pierde novedades — quedan esperando.

---

## Cómo funciona

```
    ┌─────────┐                                        ┌─────────┐
    │ Colegio │                                        │   RPI   │
    └────┬────┘                                        └────┬────┘
         │  GET /api/v1/eventos                             │
         ├─────────────────────────────────────────────────►│
         │  { eventos: [ {secuencia: 41, ...}, ... ] }      │
         │◄─────────────────────────────────────────────────┤
         │                                                  │
         │  (procesa las novedades de su lado)              │
         │                                                  │
         │  POST /api/v1/eventos/acuse  { "hasta": 43 }     │
         ├─────────────────────────────────────────────────►│
         │  { "acusados": 3 }                               │
         │◄─────────────────────────────────────────────────┤
```

Tres reglas que conviene tener claras antes de leer el detalle:

1. **Una novedad se entrega hasta que el Colegio la acusa.** Si el Colegio
   consulta y no acusa, la próxima consulta devuelve lo mismo. No se pierde nada
   por un error de red, un reinicio o un despliegue a mitad de camino.
2. **El acuse es un `POST` aparte, no un parámetro de la consulta.** Una
   consulta no cambia datos: los proxies la repiten y los clientes la reintentan
   solos, y si la consulta además marcara recibido, cada reintento perdería
   novedades en silencio.
3. **El acuse es acumulativo e idempotente.** `{"hasta": 43}` acusa todo lo que
   tenga secuencia menor o igual a 43. Mandarlo dos veces deja el mismo estado
   que mandarlo una.

### Con qué frecuencia consultar

No hay un límite impuesto, pero el ritmo útil es **cada 5 minutos**. El
procesamiento registral se mide en días; consultar cada pocos segundos no
adelanta nada y gasta de los dos lados. Si una respuesta trae `hayMas: true`,
conviene volver a consultar enseguida en lugar de esperar el próximo ciclo.

---

## Autenticación

La misma que para enviar testimonios (ver [03](03-endpoint-api.md)): **Bearer
token** estático, provisto por el RPI al Colegio.

```
Authorization: Bearer <token>
```

Sin el header, o con un token que el RPI no reconoce, los tres endpoints
responden **401**.

### Alcance: cada token ve lo que se envió con él

La lista de novedades de un token contiene **únicamente las novedades de los
testimonios que se enviaron con el canal de ese token**. No hay forma de leer
novedades de otro canal, ni de consultar un testimonio que no entró por el
canal propio: un `identificadorEnvio` de otro canal responde **404**, no 403,
para no confirmar que existe.

Si el token del Colegio se rota, el alcance **no se pierde**: está atado al
canal, no al token concreto.

---

## `GET /api/v1/eventos`

Devuelve las novedades pendientes de acuse, en orden, hasta 100 por llamada.

### Pedido

```http
GET /api/v1/eventos HTTP/1.1
Host: [URL_DEL_RPI]
Authorization: Bearer <token>
Accept: application/json
```

Sin parámetros. No hay cursor que el Colegio tenga que guardar: el RPI sabe qué
está pendiente porque el Colegio lo acusa.

### Respuesta — 200 OK

```json
{
  "eventos": [
    {
      "secuencia": 41,
      "tipo": "entrada_general_asignada",
      "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
      "ocurridoEn": "2026-06-15T10:30:15-03:00",
      "datos": {
        "entradaGeneral": {
          "numero": 12345,
          "anio": 2026,
          "numeroPresentacion": 1
        }
      }
    },
    {
      "secuencia": 42,
      "tipo": "inscripcion_definitiva",
      "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
      "ocurridoEn": "2026-06-16T14:30:22-03:00",
      "datos": {
        "entradaGeneral": {
          "numero": 12345,
          "anio": 2026,
          "numeroPresentacion": 1
        },
        "fechaSalida": "2026-06-16",
        "documento": {
          "referencia": "eg/2026/12345/1/definitiva",
          "sha256": "a511656d1e38560b9f2c...",
          "paginas": 2
        }
      }
    }
  ],
  "hayMas": false
}
```

| Campo | Tipo | Significado |
|---|---|---|
| `secuencia` | entero | Orden de la novedad. Estrictamente creciente **dentro de una respuesta**. Es el número que se acusa |
| `tipo` | texto | Ver [Tipos de novedad](#tipos-de-novedad) |
| `identificadorEnvio` | UUID | El del envío original |
| `ocurridoEn` | ISO 8601 con zona | Cuándo ocurrió el hecho del lado del RPI, no cuándo se consultó |
| `datos` | objeto | Depende del `tipo` |
| `hayMas` | booleano | `true` si quedaron novedades afuera por el límite de 100 |

Cuando no hay nada pendiente, la respuesta es `{"eventos": [], "hayMas": false}`
con **200**, no 204 ni 404.

> **Sobre `secuencia`.** Es creciente dentro de una respuesta y sirve para
> acusar, pero **no es un cursor que el Colegio deba persistir ni comparar entre
> respuestas**. Puede haber saltos (números que nunca se ven) y una novedad con
> secuencia menor puede aparecer en una respuesta posterior. Lo único que hay
> que hacer con ella es mandarla en el acuse.

---

## `POST /api/v1/eventos/acuse`

Marca como recibidas las novedades hasta la secuencia indicada.

### Pedido

```http
POST /api/v1/eventos/acuse HTTP/1.1
Host: [URL_DEL_RPI]
Authorization: Bearer <token>
Content-Type: application/json

{ "hasta": 43 }
```

| Campo | Tipo | Obligatorio | Significado |
|---|---|---|---|
| `hasta` | entero ≥ 1 | sí | Acusa todas las novedades del canal con secuencia ≤ este valor |

### Respuesta — 200 OK

```json
{ "acusados": 3 }
```

`acusados` es cuántas novedades pasaron a estado recibido con esta llamada.
Puede ser **0** y no es un error: significa que ya estaban acusadas.

### Qué acusar

Lo normal es acusar la **secuencia más alta que se procesó con éxito**. Si el
Colegio procesó las novedades 41 y 42 pero falló con la 43, tiene que acusar
`{"hasta": 42}`: la 43 vuelve en la próxima consulta.

> ⚠️ **Acusar es irreversible.** Una novedad acusada no vuelve a aparecer en la
> lista. Si el Colegio pierde una novedad ya acusada, la recupera con
> [`GET /api/v1/testimonios/{identificadorEnvio}`](#get-apiv1testimoniosidentificadorenvio),
> que informa el estado actual completo sin consumir nada. Por eso conviene
> acusar **después** de haber persistido la novedad, no antes.

---

## Tipos de novedad

| `tipo` | Cuándo | Trae documento |
|---|---|---|
| `entrada_general_asignada` | El registro asignó la Entrada General: el testimonio ingresó con su prioridad | No |
| `inscripcion_definitiva` | El trámite salió inscripto definitivamente | Sí |
| `inscripcion_provisoria` | El trámite salió observado: hay observaciones a subsanar dentro de un plazo | Sí |
| `devuelto_sin_inscribir` | El trámite salió sin inscribir | Sí |
| `inscripcion_complementaria` | Apareció un documento adicional de una Entrada General ya informada | Sí |

**Las validaciones de la recepción no generan novedades**: su resultado viaja en
la respuesta HTTP del envío (202 o error 4xx). Ver
[07](07-respuestas-y-errores.md).

> **`rechazo_registral` ya no existe como tipo.** En la versión anterior
> nombraba un rechazo por causal grave. El circuito registral real no emite esa
> figura por este canal: lo que existe es el **devuelto sin inscribir**, que es
> una salida con su documento y no un rechazo del envío. El tipo nuevo se llama
> `devuelto_sin_inscribir` justamente para no sugerir que el testimonio fue
> rechazado.

### `entrada_general_asignada`

```json
{
  "secuencia": 41,
  "tipo": "entrada_general_asignada",
  "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
  "ocurridoEn": "2026-06-15T10:30:15-03:00",
  "datos": {
    "entradaGeneral": { "numero": 12345, "anio": 2026, "numeroPresentacion": 1 }
  }
}
```

Es la primera novedad de todo testimonio aceptado, y la más importante para el
escribano: confirma que el trámite entró con su prioridad registral.

> **`fechaPresentacion` no se informa**, por el mismo motivo que en la respuesta
> de recepción ([03](03-endpoint-api.md)): de esa fecha depende la prioridad, y
> el RPI no informa una que podría estar corrida un día hábil. La fecha de la
> presentación figura en el documento de resultado cuando sale.

### `inscripcion_definitiva`

```json
{
  "secuencia": 42,
  "tipo": "inscripcion_definitiva",
  "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
  "ocurridoEn": "2026-06-16T14:30:22-03:00",
  "datos": {
    "entradaGeneral": { "numero": 12345, "anio": 2026, "numeroPresentacion": 1 },
    "fechaSalida": "2026-06-16",
    "documento": {
      "referencia": "eg/2026/12345/1/definitiva",
      "sha256": "a511656d1e38560b9f2c...",
      "paginas": 2
    }
  }
}
```

### `inscripcion_provisoria`

```json
{
  "secuencia": 57,
  "tipo": "inscripcion_provisoria",
  "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
  "ocurridoEn": "2026-06-16T11:15:00-03:00",
  "datos": {
    "entradaGeneral": { "numero": 12345, "anio": 2026, "numeroPresentacion": 1 },
    "fechaSalida": "2026-06-16",
    "vencimiento": "2026-09-22",
    "documento": {
      "referencia": "eg/2026/12345/1/provisional",
      "sha256": "d312fb8a768744fb10e7...",
      "paginas": 3
    }
  }
}
```

`vencimiento` es la fecha hasta la cual la inscripción provisional mantiene la
prioridad. **Las observaciones a subsanar están en el documento**, no en el
JSON: ver la nota de abajo.

**Cómo se subsana.** El testimonio conserva su Entrada General y su prioridad si
se subsana antes del vencimiento. La subsanación se presenta por mesa de
entradas citando la Entrada General, y el registro la incorpora como una
**presentación nueva sobre la misma Entrada General** — que después genera sus
propias novedades, con `numeroPresentacion` incrementado. La subsanación por
testimonio digital está prevista para una versión futura del contrato.

### `devuelto_sin_inscribir`

```json
{
  "secuencia": 58,
  "tipo": "devuelto_sin_inscribir",
  "identificadorEnvio": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "ocurridoEn": "2026-06-18T09:40:00-03:00",
  "datos": {
    "entradaGeneral": { "numero": 12346, "anio": 2026, "numeroPresentacion": 1 },
    "fechaSalida": "2026-06-18",
    "documento": {
      "referencia": "eg/2026/12346/1/sin-inscribir",
      "sha256": "b8bb283add7f57f6c41a...",
      "paginas": 2
    }
  }
}
```

### `inscripcion_complementaria`

Mismo formato que `inscripcion_definitiva`. Se emite cuando el registro produce
un documento adicional para una Entrada General cuyo resultado ya se informó
—por ejemplo, una plancha que se carga después de la salida—. No reemplaza al
documento anterior: lo complementa.

---

## El documento de resultado

Las cuatro novedades de salida traen un objeto `documento`:

| Campo | Significado |
|---|---|
| `referencia` | Identificador opaco para descargarlo. **No construirlo a mano**: usar el que viene en la novedad |
| `sha256` | Huella del PDF, para verificar la descarga |
| `paginas` | Cuántas páginas tiene |

> ### Lo que el documento contiene, y por qué el JSON no lo repite
>
> El documento es un **PDF firmado** que transcribe, palabra por palabra, el
> texto que el registrador redactó: las planchas de inscripción y los volantes,
> uno por página, igual que en papel. Ese texto **es el contenido jurídico del
> resultado**.
>
> Por eso este capítulo **no promete campos estructurados** con las matrículas
> inscriptas ni con las observaciones codificadas. La versión anterior los
> prometía (`matriculas`, `observaciones` con códigos) y no se puede sostener:
> el texto lo escribe una persona en el sistema registral, con plantillas a
> veces y a mano otras, y extraerle campos con un analizador sería inventar una
> precisión que el dato no tiene. Lo que sí viaja estructurado es lo que tiene
> una columna propia en el sistema registral: la Entrada General, la
> presentación, la fecha de salida y el vencimiento de la provisional.
>
> El documento incluye además un **bloque de vinculación** con el
> `identificadorEnvio` y las huellas SHA-256 del XML y del PDF del testimonio
> original. Como ese bloque queda dentro del PDF firmado, el resultado queda
> atado de forma verificable a ese testimonio exacto, y el testimonio del
> escribano **nunca se modifica**. Es el equivalente digital de abrochar la
> plancha al testimonio.

**La descarga queda registrada** del lado del RPI, con fecha, hora y canal.

> ⚠️ **Pendiente de acuerdo con el Colegio**: si el resultado se entrega como un
> PDF con todas las planchas o como un PDF por titular. Este capítulo describe
> la primera forma.

---

## `GET /api/v1/testimonios/{identificadorEnvio}`

El estado actual de un testimonio. **No consume novedades**: se puede llamar
cuantas veces se quiera y siempre informa lo mismo mientras nada cambie.

Para qué sirve:

- **Reconciliar.** Si el Colegio perdió una novedad ya acusada, o quiere
  verificar que su estado coincide con el del RPI.
- **Consultar puntualmente**, cuando un escribano pregunta por un trámite.

### Pedido

```http
GET /api/v1/testimonios/550e8400-e29b-41d4-a716-446655440000 HTTP/1.1
Host: [URL_DEL_RPI]
Authorization: Bearer <token>
```

### Respuesta — 200 OK

```json
{
  "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
  "estado": "con_resultado",
  "recibidoEn": "2026-06-15T09:12:00-03:00",
  "entradaGeneral": { "numero": 12345, "anio": 2026, "numeroPresentacion": 1 },
  "resultado": {
    "tipo": "inscripcion_definitiva",
    "fechaSalida": "2026-06-16",
    "documento": {
      "referencia": "eg/2026/12345/1/definitiva",
      "sha256": "a511656d1e38560b9f2c...",
      "paginas": 2
    }
  }
}
```

| `estado` | Significado |
|---|---|
| `aceptado` | El RPI lo recibió y validó. Todavía no tiene Entrada General |
| `con_entrada_general` | Tiene Entrada General asignada. En trámite |
| `con_resultado` | El trámite salió. `resultado` trae el tipo y el documento |

`entradaGeneral` es `null` mientras el estado sea `aceptado`; `resultado` es
`null` mientras no haya salida.

> **Este endpoint no informa el estado interno del procesamiento del RPI.** Que
> un testimonio esté sincronizándose, o esperando un reintento, es asunto del
> RPI. Para el Colegio hay tres estados y son los de arriba.

### 404

Un `identificadorEnvio` que no existe, **o que no entró por el canal del
token**, responde 404 con el formato de error de
[07](07-respuestas-y-errores.md).

---

## Errores

Los tres endpoints usan el formato de error del capítulo
[07](07-respuestas-y-errores.md).

| Código | Cuándo |
|---|---|
| 400 | `hasta` ausente, no entero o menor que 1 (solo en el acuse) |
| 401 | Falta el `Authorization`, o el token no se reconoce |
| 404 | El `identificadorEnvio` no existe o no es del canal del token |
| 429 | Demasiados pedidos. Respetar `Retry-After` |
| 503 | El RPI no puede responder en este momento. Es transitorio: reintentar |

Un 5xx o un timeout **no pierde novedades**: si no se acusó, siguen pendientes.

---

## Pendiente de acuerdo

1. **El momento con efectos legales.** Con este diseño, el momento verificable
   es la **descarga del documento**, que el RPI registra. Falta la confirmación
   formal de que ese es el momento que corre el plazo de la inscripción
   provisional. Es una definición legal, no técnica.
2. **Un PDF con todas las planchas, o uno por titular.**
3. **La cadencia de consulta**, si el Colegio necesita comprometer una distinta
   de los 5 minutos sugeridos.

---

## Próximos pasos

- Para términos del dominio que pueden no resultar familiares, andá a
  [09 — Glosario](09-glosario.md).

---

[← Índice de la documentación](README.md)
