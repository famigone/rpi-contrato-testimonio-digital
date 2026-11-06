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
         │  POST /eventos/acuse {"secuencias":[41,42,43]}   │
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
3. **El acuse lleva la LISTA de lo que se recibió**, no un tope. Mandarlo dos
   veces deja el mismo estado que mandarlo una.

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

Sin el header, o con un token que el RPI no reconoce, los cuatro endpoints
responden **401**.

### Comparten el límite de pedidos con el envío

**Sí: el presupuesto es uno por clave, para todo `/api/v1`.** Son **600 pedidos
por hora** con una ráfaga de hasta **60**; pasado el límite, **429** con
`Retry-After`.

A la cadencia sugerida no es una restricción: una consulta más un acuse cada
cinco minutos son **24 pedidos por hora**, un 4 % del presupuesto. Lo que sí
conviene tener en cuenta es que **un consultador descontrolado gastaría la cuota
de los envíos**, porque la bolsa es la misma. Si el Colegio necesita una cadencia
mucho más alta, conviene acordarla antes y no descubrirla con un 429 en medio de
un envío.

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
          "referencia": "3f9a1c04-7b2e-4d81-9f55-0a6c2e8b1d33",
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

**`entradaGeneral` es el MISMO objeto que devuelve la recepción** en
[03](03-endpoint-api.md): `numero`, `anio` (cuatro dígitos) y
`numeroPresentacion`, con los mismos nombres y los mismos tipos. No hay dos
formas de la EG en este contrato, y un cliente puede usar el mismo tipo de dato
para las dos.

Cuando no hay nada pendiente, la respuesta es `{"eventos": [], "hayMas": false}`
con **200**, no 204 ni 404.

> **Sobre `secuencia`.** Es creciente dentro de una respuesta y sirve para
> acusar, pero **no es un cursor que el Colegio deba persistir ni comparar entre
> respuestas**. Puede haber saltos (números que nunca se ven) y una novedad con
> secuencia menor puede aparecer en una respuesta posterior — que es exactamente
> por lo que el acuse lleva una lista y no un tope. Lo único que hay que hacer
> con ella es devolverla en el acuse.

---

## `POST /api/v1/eventos/acuse`

Marca como recibidas **las novedades que el Colegio enumera**.

### Pedido

```http
POST /api/v1/eventos/acuse HTTP/1.1
Host: [URL_DEL_RPI]
Authorization: Bearer <token>
Content-Type: application/json

{ "secuencias": [41, 42, 43] }
```

| Campo | Tipo | Obligatorio | Significado |
|---|---|---|---|
| `secuencias` | lista de 1 a 100 enteros ≥ 1 | sí | Las secuencias que se recibieron y procesaron |

Los duplicados se aceptan y se colapsan. Una lista vacía, un elemento que no sea
un entero ≥ 1, o más de 100 elementos, dan **400**.

> ### Por qué una lista y no un `{"hasta": N}`
>
> Un tope parece más cómodo y pierde novedades. "Acusá todo lo que sea ≤ N"
> incluye novedades que el Colegio **nunca recibió**: la lista se sirve ordenada
> por secuencia, y una novedad puede quedar disponible con una secuencia menor
> que otra que ya se entregó. Con un tope, esa novedad se marcaba recibida sin
> haberse entregado nunca, en silencio y sin que ningún error lo delatara.
>
> Enumerando, lo único que se marca recibido es lo que el Colegio dijo que
> recibió. Si una novedad no está en la lista, sigue pendiente y vuelve en la
> próxima consulta.

### Respuesta — 200 OK

```json
{ "acusados": 3 }
```

`acusados` es cuántas novedades pasaron a estado recibido con esta llamada.
Puede ser **menos que el largo de la lista**, o **0**, y no es un error:
significa que esas ya estaban acusadas.

### Qué acusar

Las que se procesaron con éxito, y solo esas. Si el Colegio recibió 41, 42 y 43
pero falló con la 43, acusa `{"secuencias": [41, 42]}`: la 43 vuelve en la
próxima consulta.

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
| `inscripcion_provisional` | El trámite salió observado: hay observaciones a subsanar dentro de un plazo | Sí |
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
      "referencia": "3f9a1c04-7b2e-4d81-9f55-0a6c2e8b1d33",
      "sha256": "a511656d1e38560b9f2c...",
      "paginas": 2
    }
  }
}
```

### `inscripcion_provisional`

```json
{
  "secuencia": 57,
  "tipo": "inscripcion_provisional",
  "identificadorEnvio": "550e8400-e29b-41d4-a716-446655440000",
  "ocurridoEn": "2026-06-16T11:15:00-03:00",
  "datos": {
    "entradaGeneral": { "numero": 12345, "anio": 2026, "numeroPresentacion": 1 },
    "fechaSalida": "2026-06-16",
    "vencimiento": "2026-09-22",
    "documento": {
      "referencia": "a1d7e820-55c3-4e9a-bb14-7f2d9c604e18",
      "sha256": "d312fb8a768744fb10e7...",
      "paginas": 3
    }
  }
}
```

`vencimiento` es la fecha hasta la cual la inscripción provisional mantiene la
prioridad. **Las observaciones a subsanar están en el documento**, no en el
JSON: ver la nota de abajo.

**Cómo se subsana** · ⚠️ **a confirmar con Registración.** El testimonio conserva
su Entrada General y su prioridad si se subsana antes del vencimiento. La
subsanación se presenta **por mesa de entradas, en papel**, citando la Entrada
General, y el registro la incorpora como una **presentación nueva sobre la misma
Entrada General**. La subsanación por testimonio digital está prevista para una
versión futura del contrato.

Lo que falta confirmar no es el circuito en papel sino **qué hace el canal
digital con el resultado de una presentación que no entró por él**: la
subsanación llega por ventanilla, y su resultado —el que de verdad le importa al
escribano, porque dice si quedó inscripto— sale sobre la misma EG. El RPI
propone entregarlo por este canal igual, con `numeroPresentacion` incrementado:
es el mismo trámite y el mismo escribano, y no entregarlo dejaría al Colegio con
la provisional y sin su desenlace. **Queda a confirmación de Registración y del
Colegio.**

> ### Cómo el RPI ve las presentaciones posteriores
>
> Es un detalle de implementación, y se cuenta porque es la razón por la que la
> promesa de arriba se puede cumplir. El canal digital conoce la presentación
> que entró por él; las siguientes no tienen ningún registro suyo. Lo que las
> hace visibles es la **Entrada General**: una vez asignada, el RPI enumera
> todas las presentaciones de esa EG por año y número, no solo la propia. En el
> sistema registral esa búsqueda cae sobre el índice natural de la tabla
> (`año, número, presentación`), así que es una lectura acotada y no un barrido.
>
> Una EG puede llegar a tener varias presentaciones —se verificaron casos de
> hasta cinco—, y cada una produce su propia novedad.

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
      "referencia": "c48b3f16-90da-4b77-8e02-15ab7c3d9f41",
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
| `referencia` | Identificador **opaco** para descargarlo, un UUID. No tiene estructura que se pueda interpretar ni construir: se usa tal como viene |
| `sha256` | Huella del PDF, para verificar la descarga |
| `paginas` | Cuántas páginas tiene |

> ### Lo que el documento contiene, y por qué el JSON no lo repite
>
> El documento es un PDF —firmado, cuando se decida con qué; hoy **sin firma**
> y marcado, ver el aviso de abajo— que transcribe, palabra por palabra, el
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
> original. Ese bloque va dentro del documento, así que cuando el documento se
> firme el resultado va a quedar atado de forma verificable a ese testimonio
> exacto — y el testimonio del escribano **nunca se modifica**. Es el
> equivalente digital de abrochar la plancha al testimonio. Hasta que haya
> firma, el bloque identifica el testimonio pero no lo garantiza
> criptográficamente.

> ### ⚠️ Con qué se firma el documento es una decisión legal PENDIENTE
>
> Hoy el resultado en papel lo firma a mano la funcionaria que lo expide. El
> equivalente digital todavía no está definido, y las opciones en evaluación son
> tres: que **la funcionaria** lo firme con su certificado, que el RPI aplique un
> **sello del organismo**, o una forma **mixta** donde la funcionaria expide
> desde el sistema y el servicio sella dejando constancia de quién expidió.
>
> **Qué significa para el Colegio, en concreto:** la forma del documento, la
> novedad y la descarga **no cambian** con lo que se decida — la decisión está
> aislada en un paso propio del diseño. Lo que puede cambiar es *quién* firma y,
> con la primera opción, que un resultado pase un rato en estado "pendiente de
> firma" antes de generar su novedad.
>
> **Mientras no haya decisión, el documento sale SIN FIRMA**, con una marca
> visible en cada página: `DOCUMENTO DE PRUEBA – SIN FIRMA DIGITAL`. No es un
> certificado de prueba ni una firma provisoria: no lleva firma digital de
> ninguna clase. Un documento con esa marca **no tiene validez registral** y no
> debe entregarse al escribano.

### `GET /api/v1/documentos/{referencia}`

```http
GET /api/v1/documentos/3f9a1c04-7b2e-4d81-9f55-0a6c2e8b1d33 HTTP/1.1
Host: [URL_DEL_RPI]
Authorization: Bearer <token>
Accept: application/pdf
```

Respuesta **200** con el PDF:

```
Content-Type: application/pdf
Content-Disposition: attachment; filename="eg-12345-2026-1-definitiva.pdf"
Content-Length: 16668
```

| Código | Cuándo |
|---|---|
| 200 | El PDF |
| 401 | Falta el `Authorization`, o el token no se reconoce |
| 404 | La referencia no existe, **o el documento es de otro canal** |
| 503 | El documento existe pero no se pudo recuperar del almacenamiento. Transitorio: reintentar |

**El 404 cubre los dos casos a propósito.** Una referencia de otro canal se
contesta igual que una inexistente: un 403 confirmaría que existe.

**Verificar la descarga** comparando el SHA-256 del PDF recibido con el `sha256`
que trajo la novedad. Si no coinciden, reintentar; no procesar el documento.

**Cada descarga queda registrada** del lado del RPI —fecha, hora, canal, clave
y resultado, incluidos los intentos fallidos—. Ese registro es el que sostiene
el argumento del momento con efectos legales: si la descarga es el hecho que
corre el plazo, tiene que haber constancia de cuándo ocurrió.

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
      "referencia": "3f9a1c04-7b2e-4d81-9f55-0a6c2e8b1d33",
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

Los cuatro endpoints usan el formato de error del capítulo
[07](07-respuestas-y-errores.md). La descarga del documento responde el PDF en
el caso bueno, y este formato en los malos.

| Código | Cuándo |
|---|---|
| 400 | `secuencias` ausente, vacía, con un elemento que no sea entero ≥ 1, o con más de 100 (solo en el acuse) |
| 401 | Falta el `Authorization`, o el token no se reconoce |
| 404 | El `identificadorEnvio` o la `referencia` no existen, **o no son del canal del token** |
| 429 | Demasiados pedidos: se agotó el presupuesto compartido de `/api/v1`. Respetar `Retry-After` |
| 503 | El RPI no puede responder en este momento, o no pudo recuperar el documento del almacenamiento. Es transitorio: reintentar |

Un 5xx o un timeout **no pierde novedades**: si no se acusó, siguen pendientes.

---

## Pendiente de acuerdo

Cinco, y las dos primeras son legales y no técnicas.

1. **El momento con efectos legales.** Con este diseño, el momento verificable
   es la **descarga del documento**, que el RPI registra. Falta la confirmación
   formal de que ese es el momento que corre el plazo de la inscripción
   provisional.
2. **Con qué se firma el documento de resultado**: la firma de la funcionaria,
   un sello del organismo, o una forma mixta. Nada de lo que este capítulo
   describe cambia con lo que se decida; ver el aviso en *El documento de
   resultado*.
3. **Qué hace el canal digital con el resultado de una presentación que entró
   por mesa de entradas** (la subsanación en papel). El RPI propone entregarlo
   por este canal igual, con `numeroPresentacion` incrementado. — Registración y
   el Colegio.
4. **Un PDF con todas las planchas, o uno por titular.** — El Colegio.
5. **La cadencia de consulta**, si el Colegio necesita comprometer una distinta
   de los 5 minutos sugeridos, teniendo en cuenta que el presupuesto de pedidos
   es compartido con el envío.

---

## Próximos pasos

- Para términos del dominio que pueden no resultar familiares, andá a
  [09 — Glosario](09-glosario.md).

---

[← Índice de la documentación](README.md)
