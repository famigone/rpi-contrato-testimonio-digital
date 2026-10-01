# 02 — Flujo end-to-end

## Diagrama de secuencia

```
Escribano → Colegio    : confecciona y firma el testimonio
Colegio   → RPI        : POST /testimonios (XML + PDF firmados)
RPI                    : valida contrato, firmas, hash, reglas del acto y tasa
RPI       → Colegio    : 202 Accepted   (o 4xx: rechazo estructural)
RPI       → Registro   : sincroniza los datos
Registro               : asigna la Entrada General
RPI       → Colegio    : callback entrada_general_asignada
Registro               : califica el documento
RPI       → Colegio    : callback inscripcion_definitiva | inscripcion_provisoria | rechazo_registral
Colegio   → Escribano  : muestra el resultado
```

## Descripción paso a paso

### 1. El escribano confecciona el testimonio

El escribano, logueado en el sistema del Colegio, completa los datos de **uno o
más actos**: por cada acto, su código, sus partes, inmuebles, monto,
certificaciones y visado; y a nivel testimonio, el otorgamiento, el rogante y el
cuerpo de la escritura. El sistema del Colegio genera:

- Un **XML** estructurado con los datos del testimonio, validable contra el XSD
  del contrato.
- Un **PDF** del testimonio completo (cuerpo de la escritura, cláusulas,
  identificación de partes, etc.).

### 2. El escribano firma digitalmente

El escribano firma los dos documentos:

- **El PDF**, con firma PAdES. El PDF firmado es el testimonio: el documento con
  valor legal.
- **El XML**, con firma XML-DSig. Protege los datos estructurados del envío y
  liga el PDF mediante su hash.

Detalles en [05 — Firma digital](05-firma-digital.md) y
[06 — Adjunto PDF](06-adjunto-pdf.md).

### 3. El sistema del Colegio envía el testimonio al RPI

`POST` al endpoint del RPI con el XML firmado y el PDF firmado, en
`multipart/form-data`. Detalles en [03 — Endpoint API](03-endpoint-api.md).

### 4. El RPI valida, persiste y responde

Todas las validaciones se hacen **antes de responder**, para que un problema
llegue al Colegio en la misma respuesta HTTP:

1. El XML está bien formado y cumple el XSD.
2. **Idempotencia**: si ya recibió un envío con ese `IdentificadorEnvio`,
   devuelve el resultado anterior sin reprocesar (ver [Idempotencia](#idempotencia)).
3. La firma XML-DSig es válida y el certificado del firmante es vigente, no está
   revocado y corresponde al escribano autorizante declarado.
4. El PDF está firmado, su firma es válida, y su hash SHA-256 coincide con el
   declarado en el XML (`MetadatosEnvio/HashPDF`).
5. Reglas de negocio: el código de acto está habilitado, las partes y datos
   corresponden al acto, el escribano y el rogante están registrados, las
   certificaciones están vigentes.
6. **Tasa registral**: el RPI consulta al sistema de tasas que la tasa declarada
   exista, esté paga y no haya sido usada en otro trámite.
7. Persiste el testimonio y almacena el PDF.
8. Responde **HTTP 202 Accepted**.

Si alguna validación falla, el RPI responde con un error 4xx y el testimonio
**no ingresa** (ver [Qué pasa si algo falla](#qué-pasa-si-algo-falla)).

**Excepción — tasa pendiente de verificación**: si el sistema de tasas no
responde a tiempo, el RPI acepta el testimonio con estado
`aceptado_tasa_pendiente` y verifica la tasa después. El testimonio no avanza al
paso 5 hasta que la tasa se confirma. Si resulta impaga, el RPI lo rechaza con el
callback `validacion_fallida`.

### 5. El RPI ingresa el testimonio al sistema registral

De manera asincrónica, el servicio del RPI sincroniza los datos con el sistema
registral, que asigna la **Entrada General (EG)**. Con la EG, el testimonio
queda ingresado al registro con su prioridad. El RPI lo notifica con el callback
`entrada_general_asignada`.

### 6. El registro califica el documento

El calificador evalúa el testimonio. El resultado posible es:

- **Inscripción definitiva**: el trámite se inscribe.
- **Inscripción provisoria**: el documento tiene observaciones que deben
  subsanarse dentro de un plazo (volante de subsanación). Conserva su prioridad
  mientras se subsana en término.
- **Rechazo registral**: el trámite no se inscribe (causales graves).

### 7. El RPI notifica al Colegio

Cada resultado se notifica por callback. El sistema del Colegio actualiza su base
interna y muestra el resultado al escribano. Detalles en
[08 — Notificaciones de callback](08-notificaciones-callback.md).

## Qué pasa si algo falla

La frontera es la **asignación de la Entrada General**. Lo que se detecta antes
es un problema del envío; lo que se detecta después es un problema registral.

### Rechazo estructural (antes de la EG)

El envío no cumple las condiciones para ingresar: el XML no respeta el contrato,
una firma no valida, la tasa no está paga, el acto no está habilitado, etc.

- Llega en la respuesta HTTP como error 4xx. Excepcionalmente (tasa verificada
  después), llega como callback `validacion_fallida`.
- El testimonio **no ingresó**: no tiene EG ni prioridad.
- **Un rechazo es definitivo para ese `IdentificadorEnvio`.** El Colegio corrige
  y hace **un envío nuevo, con un `IdentificadorEnvio` nuevo**. Para el RPI, el
  envío corregido no tiene relación con el rechazado.

### Problema transitorio del RPI

Errores 5xx, timeouts o caídas de red **no son rechazos**: el envío no tiene nada
que corregir. El Colegio reintenta el **mismo envío, con el mismo
`IdentificadorEnvio`** (ver [07 — Política de reintentos](07-respuestas-y-errores.md#política-de-reintentos)).

### Observación registral (después de la EG)

El testimonio ingresó y el calificador lo observó: llega el callback
`inscripcion_provisoria` con las observaciones y el plazo para subsanar.

- El testimonio **conserva su EG y su prioridad** mientras se subsana en término.
- **Por ahora, la subsanación se presenta por mesa de entradas**, citando la EG,
  como cualquier documento observado. El registro la incorpora como una nueva
  presentación sobre la misma EG.
- La subsanación por testimonio digital está prevista para una versión futura del
  contrato.

Una escritura rectificatoria o complementaria **no es una subsanación**: es un
documento nuevo, que se envía como un testimonio nuevo con su propio ingreso.

### Rechazo registral (después de la EG)

Llega el callback `rechazo_registral`. El trámite no se inscribe.

## Tiempos esperados

| Evento | Tiempo |
|--------|--------|
| Respuesta al POST | Típico: menos de 5 segundos. Máximo: 30 segundos |
| Asignación de la Entrada General | Minutos después del 202 |
| Calificación (inscripción o rechazo) | Variable según la carga del RPI (horas a días hábiles) |
| Callback al Colegio | Segundos después de cada evento |

Estos tiempos son orientativos.

**Recomendación para el sistema del Colegio**: no hacer esperar al escribano con
la respuesta del RPI. Registrar el envío como "enviando", devolverle el control y
hacer la llamada al RPI en un proceso en segundo plano que guarde el resultado y
reintente si hace falta.

## Idempotencia

Cada envío lleva un `IdentificadorEnvio` (UUID) generado por el Colegio. La
regla es:

- **Mismo envío → mismo identificador.** Si el Colegio reenvía exactamente el
  mismo testimonio (por un timeout, un error de red o un 5xx), el RPI devuelve el
  resultado del envío original sin reprocesarlo: el estado actual si fue
  aceptado, o el mismo error si fue rechazado.
- **Envío distinto → identificador nuevo.** Un envío corregido después de un
  rechazo es un envío distinto. Si se reusa el identificador con contenido
  distinto, el RPI responde `409 IDENTIFICADOR_DUPLICADO_CONTENIDO_DISTINTO`.

Esto permite reintentar sin riesgo de duplicación, y obliga a que cada corrección
sea un envío identificable.

## Próximos pasos

- Para el contrato técnico del endpoint, seguí con
  [03 — Endpoint API](03-endpoint-api.md).
- Para el formato del XML, andá a [04 — Formato XML](04-formato-xml.md).

---

[← Índice de la documentación](README.md)