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
| 400 Bad Request | **Todo rechazo del envío**: XML mal formado o inválido (XSD), reglas del contrato no cumplidas, multipart inválido, hash del PDF, y **cualquier problema de firma o de certificado** | Corregir y enviar con un `identificadorEnvio` nuevo. |
| 401 Unauthorized | **Solo la clave de API**: faltante o inválida | Revisar la clave. No es un problema del envío. |
| 403 Forbidden | *Reservado, no implementado.* Hoy no se devuelve nunca | — |
| 409 Conflict | *Reservado, no implementado.* Un `identificadorEnvio` repetido devuelve **200** con `yaProcesadoAnteriormente: true`, sin comparar el contenido | — |
| 413 Payload Too Large | Petición excede el límite (50 MB total) | Reducir el tamaño del PDF y enviar con un `identificadorEnvio` nuevo. |
| 422 Unprocessable Entity | *Reservado, no implementado.* Las reglas del contrato que el XSD no expresa viajan en el **400**, con `motivo: "estructura_invalida"` y la lista en `info.reglas_fallidas` | Corregir y enviar con un `identificadorEnvio` nuevo. |
| 429 Too Many Requests | Rate limit del RPI alcanzado | Reintentar con el mismo `identificadorEnvio`, respetando `Retry-After`. |
| 500 Internal Server Error | Error del lado del RPI | Reintentar con el mismo `identificadorEnvio` y backoff exponencial. |
| 502 Bad Gateway | RPI temporalmente no disponible | Ídem. |
| 503 Service Unavailable | RPI en mantenimiento o dependencia no disponible | Ídem, respetando `Retry-After` si viene. |
| 504 Gateway Timeout | *Reservado, no implementado.* Una demora termina en 503 o en 500 | — |

## Formato del cuerpo de error

El cuerpo es **plano**: `error` es un **texto**, no un objeto.

```json
{
  "error": "validacion_fallida",
  "motivo": "estructura_invalida",
  "detalle": "El XML no cumple el contrato v3",
  "info": {
    "motivo": "estructura_invalida",
    "errores": [
      { "mensaje": "Element 'Partes': Missing child element 'Parte'.", "linea": 42 }
    ]
  },
  "fechaRechazo": "2026-10-26T10:23:45.000Z"
}
```

| Campo | Qué es |
|---|---|
| `error` | La familia: `validacion_fallida` para los rechazos del envío, `tamano_excedido` en el 413, `no_autorizado` en el 401, `error_interno` en el 500 |
| `motivo` | **El dato sobre el que conviene ramificar.** Identifica el rechazo: `estructura_invalida`, `xml_malformado`, `cuit_invalido`, `firma_invalida`, `multipart_invalido`, `hash_pdf_no_coincide`, `pdf_firmante_no_coincide`, `pdf_firma_invalida`, `pdf_cadena_no_confiable`, `pdf_contenido_alterado`, `pdf_invalido`, `identificador_ausente`, `codigo_no_habilitado`, `tamano_excedido` |
| `detalle` | El mensaje en castellano, pensado para mostrarlo o registrarlo |
| `info` | Lo específico de cada motivo: `causa` cuando el motivo tiene variantes, y listas como `reglas_fallidas`, `cuits_invalidos`, `errores` o `firmantesPdf` |
| `fechaRechazo` | Cuándo se rechazó |

> **Una excepción, y es la única:** el **429** usa un sobre anidado,
> `{ "error": { "codigo": "RATE_LIMIT", "mensaje": … } }`. Conviene manejar las
> dos formas: si `error` es un objeto, leer `error.codigo`; si es un texto,
> leer `motivo`.

> **`identificadorEnvio` no viaja en el cuerpo del error.** Si hace falta
> correlacionar, el cliente ya lo tiene: es el que envió.

## Catálogo de códigos de error

### Errores de petición (4xx)

> **Cómo leer esta tabla.** La columna `Código` es el nombre con el que este
> documento se refiere a cada error. **Lo que viaja en la respuesta** es el
> cuerpo de la sección anterior: `error`, `motivo` y —cuando el motivo
> distingue variantes— `info.causa`. La última columna dice exactamente qué
> esperar, para que el cliente pueda ramificar sobre eso y no sobre el nombre.
>
> Las filas marcadas ***reservado*** están documentadas pero **el servicio no
> las devuelve hoy**. No se borran: describen comportamiento previsto, y se
> indica qué pasa en su lugar.

| Código | HTTP | Qué viaja en la respuesta | Descripción |
|--------|------|---------------------------|-------------|
| `XML_INVALIDO` | 400 | `motivo: "estructura_invalida"` | El XML no cumple el XSD. `detalle` indica línea/columna/elemento. |
| `XML_NO_PARSEABLE` | 400 | `motivo: "xml_malformado"` | El XML está mal formado (no es XML válido). |
| `CUIT_INVALIDO` | 400 | `motivo: "cuit_invalido"` | El **dígito verificador** de algún CUIT del XML no es correcto. `info.cuits_invalidos` trae la lista completa, con el campo y la parte de cada uno. |
| `FIRMA_INVALIDA` | **400** | `motivo: "firma_invalida"` + `info.causa` | La firma del XML no se puede verificar. La `causa` dice cuál de todas: `firma_no_parseable`, `firma_no_verifica`, `digest_documento_no_coincide`, `algoritmo_no_permitido`, `firma_envuelta_no_admitida`, entre otras. |
| `FIRMA_FALTANTE` | **400** | `motivo: "multipart_invalido"`, `causa: "falta_firma_escribano"` | Falta la parte `firma_xml_escribano` del multipart. Desde la **v3.1** la firma NO va dentro del XML: un `<ds:Signature>` adentro se rechaza con `causa: "firma_envuelta_no_admitida"`. |
| `CERTIFICADO_NO_VIGENTE` | **400** | `motivo: "firma_invalida"`, `causa: "cadena_no_confiable"` (XML) · `motivo: "pdf_cadena_no_confiable"` (PDF) | El certificado del firmante está vencido a la fecha de recepción. |
| `CERTIFICADO_REVOCADO` | — | ***reservado, no implementado*** | El RPI **no consulta revocación**: sin OCSP ni CRL, por decisión de diseño del **ADR-003**. Un certificado revocado pero vigente y con cadena válida hoy **pasa**. |
| `CERTIFICADO_NO_RECONOCIDO` | **400** | `motivo: "firma_invalida"`, `causa: "cadena_no_confiable"` (XML) · `motivo: "pdf_cadena_no_confiable"` (PDF) | La cadena del firmante no ancla en una raíz confiable del RPI. |
| `CUIT_NO_COINCIDE` | 400 | El CUIT del certificado del firmante no coincide con el del XML, o no se pudo leer. Viaja como `motivo: "firma_invalida"` con una `causa`: ver [El CUIT del firmante](#el-cuit-del-firmante-causas-de-firma_invalida). |
| `HASH_PDF_NO_COINCIDE` | 400 | `motivo: "hash_pdf_no_coincide"` | El hash declarado en el XML no coincide con el PDF recibido. |
| `PDF_NO_FIRMADO` | 400 | `motivo: "pdf_firmante_no_coincide"` | Ninguna firma del PDF corresponde al escribano autorizante: incluye el caso del PDF sin ninguna firma. `info.firmantesPdf` trae lo que se encontró. |
| `PDF_FIRMA_INVALIDA` | 400 | `motivo: "pdf_firma_invalida"` · `"pdf_contenido_alterado"` | La firma del escribano en el PDF no verifica, o no cubre todo el documento. |
| `MULTIPART_INVALIDO` | 400 | La petición multipart está mal formada, falta una parte, o **trae dos partes con el mismo nombre**. |
| `IDENTIFICADOR_INVALIDO` | 400 | `motivo: "identificador_ausente"` (falta) · `"estructura_invalida"` (formato) | El `IdentificadorEnvio` falta o no tiene el formato del XSD. El formato lo valida el XSD, así que llega como error de estructura. |
| `IDENTIFICADOR_DUPLICADO_CONTENIDO_DISTINTO` | — | ***reservado, no implementado*** | Hoy un `identificadorEnvio` repetido devuelve **200** con `yaProcesadoAnteriormente: true` y el estado del envío original, **sin comparar el contenido**. |
| `AUTENTICACION_REQUERIDA` | 401 | `{ "error": "no_autorizado" }` | Falta la clave de API. |
| `AUTENTICACION_INVALIDA` | 401 | `{ "error": "no_autorizado" }` | La clave de API no es válida. **Mismo cuerpo que el caso anterior**, a propósito: no se distingue "falta" de "es inválida". |
| `SIN_PERMISO` | — | ***reservado, no implementado*** | No hay permisos por cliente: una clave válida puede enviar testimonios. |
| `LIMITE_TAMANO` | 413 | `error: "tamano_excedido"`, `motivo: "tamano_excedido"` | La petición supera el límite. `info.campo` dice cuál parte lo superó y `info.maxBytes` el tope. |
| `RATE_LIMIT` | 429 | `{ "error": { "codigo": "RATE_LIMIT", … } }` | Demasiadas peticiones del mismo cliente. Respetar `Retry-After`. **Es el único error cuyo cuerpo tiene el sobre `error.codigo`** y no el `error`/`motivo` del resto. |

---

### `CUIT_INVALIDO` — el dígito verificador

El XSD valida la **forma** del CUIT (`\d{2}-?\d{8}-?\d{1}`, con o sin
guiones) y el RPI valida el **dígito verificador** por módulo 11. Se revisan
**todos** los CUIT del testimonio:

- `EscribanoAutorizante/CUIT`
- `Rogante/CUIT`
- el `CUIT` de cada `Parte`
- el `CUIT` de cada `Representante`

**El prefijo no importa.** Un `30` de persona jurídica es tan válido como un
`20` o un `27`: el dígito verificador no dice nada del tipo de persona, y el RPI
no restringe el prefijo en ningún lado.

Se informan **todos** los que fallan, no el primero: si hay tres mal cargados,
el detalle trae los tres y no hace falta reenviar tres veces para descubrirlos
de a uno.

```json
{
  "error": "validacion_fallida",
  "motivo": "cuit_invalido",
  "detalle": "El dígito verificador no es correcto en 2 CUIT del testimonio",
  "info": {
    "motivo": "cuit_invalido",
    "cuits_invalidos": [
      { "campo": "EscribanoAutorizante/CUIT", "valor": "20-12345678-9" },
      {
        "campo": "Acto[1]/Partes/Parte[2]/CUIT",
        "valor": "27-14567890-4",
        "acto": 1,
        "parte": 2,
        "rol": "TRANSMITENTE"
      }
    ]
  }
}
```

`campo` es la ruta del dato en el XML, y para las partes se agregan el número de
acto, la posición de la parte dentro del acto y su rol, que es como las
identifica quien lee el documento.

Un CUIT que no tenga 11 dígitos **no** da este error: eso ya lo rechaza el XSD
por forma, y duplicar el rechazo daría dos códigos para el mismo problema.

---

### `MULTIPART_INVALIDO` — dos partes con el mismo nombre

El multipart lleva exactamente tres partes: `xml`, `pdf` y
`firma_xml_escribano`. **Mandar dos veces la misma se rechaza.** Antes la
segunda sobrescribía a la primera en silencio.

Con la firma detached esto importa más que antes: la firma cubre **un** juego de
bytes, así que con dos partes `xml` cuál de las dos se verificaba quedaba a
merced del orden en que llegaran. El `detalle` trae `causa: "campo_duplicado"` y
el nombre del campo repetido.

---

### El CUIT del firmante — causas de `firma_invalida`

El RPI verifica que **quien firmó sea el escribano autorizante declarado**:
compara el CUIT del certificado con `EscribanoAutorizante/CUIT` del XML. El
control es el mismo, y **estricto**, en las dos firmas: la del XML
(`firma_xml_escribano`) y la **PAdES del PDF**.

El rechazo viaja como `motivo: "firma_invalida"` con una de dos **causas**:

| `causa` | Significa |
|---|---|
| `cuit_firmante_ilegible` | El `serialNumber` del sujeto del certificado **no trae un CUIT/CUIL de 11 dígitos**, así que no hay con qué comparar |
| `cuit_firmante_no_coincide` | Trae un CUIT, pero **es otro** |

Son dos problemas distintos: el primero es del certificado o de cómo la AC
cargó el campo, el segundo es que la firma no es de quien dice el XML.

#### De dónde se lee el CUIT, y qué formatos se aceptan

Del atributo **`serialNumber`** del *subject* del certificado (OID 2.5.4.5).
Se aceptan todas estas formas:

| Ejemplo | Qué tiene |
|---|---|
| `CUIL 20316955240` | La forma habitual de los certificados de **AC-ONTI** |
| `CUIT 30561986653` | Ídem, persona jurídica |
| `cuit 30-56198665-3` | La palabra en cualquier capitalización, y con guiones |
| `CUIT: 30561986653` | Con dos puntos, con o sin espacio |
| `CUIL20316955240` | Sin separación entre la palabra y el número |
| `CUIT 30 56198665 3` | Espacios como separadores |
| `30561986653` | El número solo, sin palabra |
| `30-56198665-3` | Ídem, con guiones |

**El prefijo no importa**: un `30` de persona jurídica se lee igual que un `20`
o un `27`.

Lo que **no** se acepta:

- Cualquier cosa que no deje **exactamente 11 dígitos** después de sacar la
  palabra y los separadores. En particular, un `CUIT 123` no es un CUIT.
- Un campo encabezado por **otra palabra de documento** (`DNI`, `LE`, `LC`,
  `CI`, `PAS`): si el certificado dice que es un DNI, el RPI no lo toma por un
  CUIT.
- La palabra `CUIT`/`CUIL` **en el medio** de otro texto (por ejemplo
  `Escribano CUIT 30561986653`). Tiene que estar al principio del campo: buscarla
  en cualquier posición haría pasar un amarre de identidad que en realidad no se
  verificó.

#### El detalle del rechazo

Informa **el número que se leyó y el que se esperaba**, los dos en
`NN-NNNNNNNN-N`:

```json
{
  "error": "validacion_fallida",
  "motivo": "firma_invalida",
  "detalle": "El CUIT del certificado firmante no coincide con el del escribano autorizante declarado",
  "info": {
    "motivo": "firma_invalida",
    "causa": "cuit_firmante_no_coincide",
    "cuitLeido": "20-31695524-0",
    "cuitEsperado": "30-56198665-3"
  },
  "fechaRechazo": "2026-10-26T12:00:00.000Z"
}
```

Y cuando no se pudo leer, `cuitLeido` es `null`:

```json
{
  "motivo": "firma_invalida",
  "causa": "cuit_firmante_ilegible",
  "cuitLeido": null,
  "cuitEsperado": "30-56198665-3"
}
```

> **El `serialNumber` crudo no se informa.** Puede traer otro documento del
> firmante, y para corregir el envío alcanza con saber que no se pudo leer un
> CUIT de ahí.

En la firma del **PDF** el rechazo es `motivo: "pdf_firmante_no_coincide"` y el
detalle trae además `firmantesPdf`, con el CUIT leído de **cada** firma del
documento: un PDF puede tener varias, y el RPI evalúa la del escribano.

---

### Errores de negocio (422) — *el 422 está reservado*

> **El RPI no devuelve 422 hoy.** Las reglas del contrato que el XSD no puede
> expresar se validan igual, pero el rechazo viaja en un **400** con
> `motivo: "estructura_invalida"` y **la lista completa** en
> `info.reglas_fallidas`, cada entrada con su `regla`, el `acto` donde falló y
> un `detalle`:
>
> ```json
> {
>   "error": "validacion_fallida",
>   "motivo": "estructura_invalida",
>   "detalle": "Reglas condicionales del contrato no cumplidas",
>   "info": {
>     "motivo": "estructura_invalida",
>     "reglas_fallidas": [
>       { "regla": "PROPORCIONES_NO_SUMAN_UNO", "acto": 1, "detalle": { "suma": "3/4" } }
>     ]
>   }
> }
> ```
>
> Los códigos de abajo son los valores de `regla`, salvo los marcados.
> Se informan **todas** las reglas que fallan, no la primera.

| Código | Descripción |
|--------|-------------|
| `ESCRIBANO_NO_REGISTRADO` | ***reservado, no implementado.*** El RPI no valida al escribano contra un catálogo en la recepción. |
| `ROGANTE_NO_ENCONTRADO` | **No es un rechazo de la recepción.** El envío se acepta; el fallo aparece después, al sincronizar con el sistema registral, si el CUIT del escribano no está dado de alta. Se resuelve con un alta manual en el RPI. |
| `MATRICULA_NO_EXISTENTE` | ***reservado, no implementado.*** La matrícula no se verifica contra el padrón en la recepción. |
| `CERTIFICACION_VENCIDA` | ***reservado, no implementado.*** No se evalúa el vencimiento de las certificaciones previas. |
| `VERSION_CONTRATO_NO_SOPORTADA` | ***reservado.*** Una versión no soportada se rechaza por el XSD, así que llega como `motivo: "estructura_invalida"`. |
| `CODIGO_ACTO_NO_HABILITADO` | El `<Codigo>` del acto existe en el catálogo pero **no está habilitado** para testimonio digital. Viaja como `motivo: "codigo_no_habilitado"` en un **400**, no como una `regla`. |
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
> **No hay errores de tasa.** Antes figuraban `TASA_INEXISTENTE`,
> `TASA_NO_PAGADA` y `TASA_YA_UTILIZADA`. **Se eliminaron:** la ley impide
> rechazar un trámite por cuestiones tributarias, así que una tasa impaga,
> inexistente o ya usada **no rechaza el ingreso**. El testimonio entra igual y
> la inscripción sale provisional hasta que se pague. Ver
> [03 — La tasa en la respuesta](03-endpoint-api.md#la-tasa-en-la-respuesta).
>
> Con eso desaparece también el único rechazo que podía llegar después del 202:
> **todo rechazo estructural es síncrono**.

### Errores del servidor (5xx)

| Código | HTTP | Descripción |
|--------|------|-------------|
| `ERROR_INTERNO` | 500 | Error inesperado del RPI. Viaja como `error: "error_interno"`, con un `requestId` para rastrearlo. Reintentar con backoff. |
| `BD_NO_DISPONIBLE` | 503 | Una dependencia del RPI no está disponible. Viaja como `error: "infra_error"`. |
| `MANTENIMIENTO` | 503 | El RPI todavía no terminó de verificar sus dependencias, o una no responde. Header `Retry-After`. |
| `VALIDACION_NO_DISPONIBLE` | 503 | **El RPI no puede validar**: le falta configuración, típicamente las raíces de confianza. El mensaje dice explícitamente que **NO** se reintente: el envío no tiene nada de malo y reintentar no va a funcionar hasta que lo arreglen en el RPI. |
| `TIMEOUT_INTERNO` | 504 | ***reservado, no implementado.*** El RPI no devuelve 504 hoy; una demora termina en 503 o en 500. |

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
| Observación registral (callback `inscripcion_provisoria`) | No aplica: se subsana por mesa de entradas (ver [02](02-flujo-end-to-end.md#observación-registral-después-de-la-eg)) |

## Ejemplos de respuestas de error

### XML inválido

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "validacion_fallida",
  "motivo": "estructura_invalida",
  "detalle": "El XML no cumple el contrato v3",
  "info": {
    "motivo": "estructura_invalida",
    "errores": [
      {
        "mensaje": "Element 'Partes': Missing child element(s). Expected is ( Parte ).",
        "linea": 42
      }
    ]
  },
  "fechaRechazo": "2026-10-26T10:23:45.000Z"
}
```

### Firma inválida

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "validacion_fallida",
  "motivo": "firma_invalida",
  "detalle": "El digest del XML no coincide con el de la firma: el documento no es el que se firmó. Atención a los finales de línea (el digest es sobre los bytes crudos, sin canonicalizar).",
  "info": {
    "motivo": "firma_invalida",
    "causa": "digest_documento_no_coincide",
    "esperado": "qPr8…",
    "calculado": "7Hs2…",
    "bytesXml": 18432
  },
  "fechaRechazo": "2026-10-26T10:23:45.000Z"
}
```

### Hash de PDF no coincide

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "validacion_fallida",
  "motivo": "hash_pdf_no_coincide",
  "detalle": "El HashPDF declarado no coincide con el SHA-256 del PDF recibido",
  "info": {
    "motivo": "hash_pdf_no_coincide",
    "hashDeclarado": "3a7bd3e2360a3d29eea436fcfb7e44c735d117c42d1c1835420b6b9942dd4f1b",
    "hashCalculado": "8e34a7c1f9b8e0c8de72e80e2b9bf28c1c34e2f0b7d5a3c1f8e9b8e0c8de72e8f"
  },
  "fechaRechazo": "2026-10-26T10:23:45.000Z"
}
```

