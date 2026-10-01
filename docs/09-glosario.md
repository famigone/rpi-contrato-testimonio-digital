# 09 — Glosario

Términos del dominio registral y notarial que aparecen en este contrato.

## Términos generales

### Acto

La operación jurídica que se inscribe en el RPI. Un testimonio puede contener
**N actos** (elemento `<Acto>` dentro de `<Actos>`). El tipo de acto se
identifica por su **`<Codigo>`** (el código del catálogo `act`: 1028 compraventa,
1075 hipoteca, 1056 donación, 1102 permuta, 1020 cancelación, etc.), no por un
elemento nombrado del esquema. Ver [`catalogo-actos.json`](../catalogo-actos.json).

### Código de acto

El número del catálogo `act` que identifica el tipo de acto. Es el elemento
`<Codigo>` del `<Acto>`. El XSD no lleva un enum de códigos: qué códigos existen
se publica en [`catalogo-actos.json`](../catalogo-actos.json); qué códigos están
*habilitados* lo decide el servicio y se publica en
[`artefacto-campos-por-acto.json`](../artefacto-campos-por-acto.json).

### Familia (de acto)

Agrupación estructural de códigos que comparten roles, montos y
certificaciones. Las familias son `TRANSMISION` (compraventa, donación, permuta,
etc.), `HIPOTECARIA_CONSTITUYE` (hipoteca, ampliación), `HIPOTECARIA_LIBERA`
(cancelación, liberación, reducción) y `DECLARATIVO` (actos con un solo sujeto y
sin montos). El servicio valida por familia qué corresponde a cada código
habilitado. Ver [11 — Artefacto de campos por acto](11-artefacto-campos-por-acto.md).

### Adquirente

Persona que **recibe** el inmueble en una transmisión. En el sistema legacy se
llama "titular adquirente". En el contrato es una `<Parte rol="ADQUIRENTE">`.

### Parte

Persona que interviene en un acto. Cada acto tiene una lista de `<Parte>`, donde
el rol se indica con el atributo `rol`. Una `Parte` es una persona (con su `Tipo`
H/J/O, representante y proporción) más el atributo `rol`.

### Rol de parte

Atributo obligatorio de cada `<Parte>` que indica su papel en el acto. Roles
definidos: `ADQUIRENTE`, `TRANSMITENTE`, `ACREEDOR`, `DEUDOR` (`ACREEDOR` y
`DEUDOR` se usan en los actos hipotecarios, tanto de constitución como de
cancelación). El XSD acepta cualquier rol en cualquier acto; la correspondencia
rol/acto la valida el servicio del RPI según la familia del código.

### Asiento

Cada anotación que se hace en una matrícula del registro. Cada vez que se
inscribe un acto sobre un inmueble, se registra un asiento nuevo.

### Calificador

Funcionario del RPI que evalúa si un trámite cumple los requisitos para ser
inscripto. Puede inscribir, observar (inscripción provisoria) o rechazar.

### Certificación registral

Certificado emitido por el RPI antes del acto, que el escribano solicita antes
de otorgar la escritura para asegurarse del estado del bien y de las personas.
En el contrato son **dos certificados distintos**, cada uno con su número y
fecha de emisión, y cada uno vive donde recae lo que certifica: la
**certificación de dominio** (`CertificacionDominio`), sobre el estado dominial
del **inmueble**, va a **nivel acto**; la **certificación de inhibición**
(`CertificacionInhibicion`), sobre si la **persona** que dispone está inhibida,
va **dentro de la `<Parte>`** correspondiente. Ambas son opcionales en el XSD;
su exigencia depende de la familia del acto (por ejemplo, la compraventa exige
ambas y la cancelación de hipoteca ninguna).

### Compraventa

Acto jurídico por el cual una parte (transmitente) transfiere el dominio de
un inmueble a otra parte (adquirente) a cambio de un precio.

### Cuerpo (de la escritura)

El texto completo de la escritura notarial, con todas sus cláusulas. Incluye
identificación de partes, descripción del inmueble, condiciones de la
operación, asentimientos, etc.

## Términos del RPI

### Barra catastral

División registral interna del inmueble. Campo opcional dentro de la
identificación del inmueble.

### Certificación catastral

Certificación del inmueble emitida por catastro, distinta de la certificación
registral. Informa el estado catastral (nomenclatura, superficie, plano) y
puede tener observaciones. Se incluye **dentro de cada inmueble**. Es opcional
en el XSD; su exigencia depende de la familia del acto (la exigen, por ejemplo,
las transmisiones y las hipotecas; no la llevan las cancelaciones).

### EG / Entrada General

Número que el RPI asigna a cada trámite cuando ingresa al registro. Es el
identificador del trámite dentro del RPI. Tiene un número y un año (ej:
12345/2026), y cada ingreso o reingreso sobre ella es una
[presentación](#presentación) numerada.

Con la Entrada General, el trámite obtiene su **prioridad registral**. En el
flujo de testimonio digital, la EG se asigna **después** de que el RPI acepta el
testimonio (HTTP 202) y se notifica con el callback `entrada_general_asignada`.

La asignación de la EG es la frontera entre los dos tipos de problema que puede
tener un testimonio: lo que se detecta antes es un
[rechazo estructural](#rechazo-estructural); lo que se detecta después, una
[observación registral](#observación-registral).

### Inscripción definitiva

Estado final del trámite cuando todo está conforme y queda anotado en la
matrícula del inmueble. Tiene plena eficacia legal.

### Inscripción provisoria

Estado del trámite cuando hay observaciones que el escribano debe subsanar
antes de la inscripción definitiva. El trámite se anota provisoriamente
mientras se subsanan los defectos. Tiene una vigencia limitada durante la cual
se mantiene el orden de prelación; el vencimiento se informa en el callback
(`fechaVencimientoVIP`).

### Matrícula

Identificador único del inmueble en el RPI. Cada inmueble tiene una matrícula
y todas las anotaciones sobre él se hacen en esa matrícula. En el contrato la
matrícula es un número entero (hasta 8 dígitos) y el departamento se identifica
por separado con un código numérico (1 a 16).

### Minuta

Documento que históricamente acompaña al testimonio en papel, conteniendo en
forma resumida los datos del acto para facilitar su carga al sistema
registral. En el flujo digital, la minuta es reemplazada por los datos
estructurados del XML.

### Nomenclatura catastral

Identificación parcelaria del inmueble. En el contrato se modela como 5 campos
de longitud fija (2, 2, 3, 4 y 4 caracteres) según la convención del RPI. Se
incluye **dentro de cada inmueble**. Como la certificación catastral, es
opcional en el XSD y su exigencia depende de la familia del acto.

### Observación registral

Defecto que el calificador encuentra en un trámite que **ya ingresó** (tiene
Entrada General). Da lugar a una
[inscripción provisoria](#inscripción-provisoria) y se notifica con el callback
`inscripcion_provisoria`. El trámite conserva su EG y su prioridad mientras se
subsana en término. No confundir con el
[rechazo estructural](#rechazo-estructural).

### Plataforma Digital (PD)

Canal digital existente del RPI para formularios estructurados (Dominio,
Inhibición, Afectación Vivienda). Convive con el canal del testimonio digital.

### Mesa de Entradas Digital (MED)

Canal actual del RPI para presentación de testimonios en papel con
acompañamiento de minuta digital. Convive con el canal del testimonio digital.

### Presentación

Cada ingreso de documentación sobre una misma Entrada General, numerado en
orden: la presentación 1 es el ingreso del trámite; las siguientes son
subsanaciones de observaciones. En los callbacks se informa como
`numeroPresentacion` y `fechaPresentacion` dentro de `entradaGeneral`.

### Rechazo estructural

Rechazo de un envío que **no llegó a ingresar** al registro: no respeta el
contrato, una firma no valida, la tasa no está paga, el acto no está
habilitado, etc. Lo detecta el servicio del RPI antes de asignar Entrada
General. Llega como error HTTP 4xx o, excepcionalmente, como callback
`validacion_fallida`. El envío no tiene EG ni prioridad: se corrige y se envía
de nuevo con un `IdentificadorEnvio` nuevo. Ver
[07 — Respuestas y errores](07-respuestas-y-errores.md).

### Rechazo registral

Cuando el calificador determina que un trámite **ya ingresado** tiene defectos
graves que no pueden subsanarse (por ejemplo, matrícula que no corresponde) y
rechaza la inscripción. Se notifica con el callback `rechazo_registral`.

### Rogante

Persona que actúa como **representante del trámite ante el RPI**. Es quien
"ruega" la inscripción. Típicamente es el escribano autorizante, pero puede
ser otra persona designada. El RPI se comunica con el rogante para
notificaciones formales del trámite. Debe estar registrado en el RPI.

### Subsanación

Presentación de lo necesario para corregir una
[observación registral](#observación-registral). Se incorpora como una nueva
[presentación](#presentación) sobre la misma Entrada General, conservando la
prioridad. Por ahora se presenta por mesa de entradas, citando la EG; la
subsanación por testimonio digital está prevista para una versión futura. Una
escritura rectificatoria o complementaria no es una subsanación: es un
documento nuevo con su propio ingreso.

### Tomo / Folio / Finca

Sistema de identificación de inmuebles previo al folio real (matrícula). Se usa
para inmuebles antiguos que aún no fueron matriculados. En el contrato son tres
campos opcionales que se completan en conjunto cuando el inmueble no tiene
matrícula.

### Tasa registral

Pago al RPI por el servicio de inscripción del trámite. Se acredita mediante
un número de tasa que el escribano gestiona antes del envío. Al recibir el
testimonio, el RPI verifica en el sistema de tasas que exista, esté paga y no
haya sido usada en otro trámite.

### Testimonio

Copia autenticada por el escribano de la escritura matriz, con valor de título
inscribible. Históricamente se presenta en papel al RPI; en el flujo digital,
el testimonio es el PDF firmado digitalmente por el escribano.

### Transmitente

Persona que **entrega** el inmueble en una transmisión. En el sistema legacy
se llama "titular transmitente". En el contrato es una
`<Parte rol="TRANSMITENTE">`.

### Visado de Rentas

Validación previa al acto por la Dirección Provincial de Rentas. En el contrato
es un bloque obligatorio: con visado se indica `Tipo=R` (y se incluye el número
de trámite); sin visado (exento u otra causal) se indica `Tipo=A`.

### VIP / VIO / Volante de Inscripción Provisoria

Documento que el RPI emite cuando un trámite queda inscripto provisoriamente,
detallando las observaciones que deben subsanarse (volante de subsanación).
Por extensión, se usa "VIP" para referirse al estado de inscripción provisoria
en sí.

## Términos notariales

### Asentimiento conyugal

Cuando el inmueble es ganancial (durante el matrimonio en régimen de comunidad),
el cónyuge de quien dispone del inmueble debe prestar asentimiento. En el
contrato se modela como **texto libre** en el campo `AsentimientoConyugal`,
hijo opcional de `<Acto>`, no como bloque estructurado. Qué rol presta el
asentimiento depende del acto (ver
[11 — Artefacto de campos por acto](11-artefacto-campos-por-acto.md)).

### Escribano autorizante

El notario que firma la escritura y da fe del acto. Es quien firma
digitalmente el PDF del testimonio y el XML del envío.

### Escritura

Documento notarial original (matriz) que queda en el protocolo del escribano.
El testimonio es una copia autenticada de la escritura.

### Escritura rectificatoria o complementaria

Escritura nueva que corrige o completa otra anterior. Es un instrumento
distinto: se envía como un testimonio nuevo, con su propio ingreso, no como
[subsanación](#subsanación).

### Folio de protocolo

Cada escritura ocupa una o varias fojas (folios) numeradas dentro del
protocolo del escribano. Se identifica por número y año.

### Organismo público

Persona pública estatal o paraestatal que puede ser parte del acto. En el
contrato se identifica con `Tipo=O`.

### Otorgamiento

El acto de firmar la escritura. Tiene lugar y fecha. El "otorgamiento" se
refiere tanto al hecho como a los datos que lo identifican.

### Persona humana

Persona física (a diferencia de persona jurídica). En el contrato se identifica
con `Tipo=H`.

### Persona jurídica

Sociedad, asociación, fundación; persona no humana. En el contrato se identifica
con `Tipo=J`. Usa `ApellidoODenominacion` como razón social.

### Proporción

En un acto con varios adquirentes, indica qué fracción del inmueble adquiere
cada uno. Se modela como string fracción (`"1/2"`, `"1/3"`, `"1/1"`) dentro de
la `<Parte>`. Si las proporciones deben sumar 1 depende del acto (ver
[11 — Artefacto de campos por acto](11-artefacto-campos-por-acto.md)); la
validación la hace el servicio del RPI.

### Representante

Persona que actúa por cuenta de otra (tutor, apoderado, etc.). En el contrato
es un bloque opcional dentro de cada `<Parte>`, cualquiera sea su rol.

### Protocolo

Libro encuadernado donde el escribano archiva las escrituras matrices.

## Términos técnicos del contrato

### Idempotencia

Propiedad de una operación que produce el mismo resultado sin importar cuántas
veces se ejecute. En este contrato, reenviar el mismo testimonio (mismo
`IdentificadorEnvio`, mismo contenido) produce siempre el mismo resultado: el
testimonio se procesa una sola vez, y el reenvío recibe el estado actual si fue
aceptado o el mismo error si fue rechazado.

### IdentificadorEnvio

UUID v4 generado por el sistema del Colegio para identificar unívocamente
cada envío. Regla: **mismo envío, mismo identificador** (reintentos por timeout,
error de red o 5xx); **envío distinto, identificador nuevo** (por ejemplo, un
envío corregido después de un rechazo). Reusar un identificador con contenido
distinto devuelve 409.

### XML-DSig

XML Digital Signature, estándar W3C para firma digital de documentos XML.
El XML del testimonio se firma con XML-DSig por el escribano autorizante.
Protege los datos estructurados del envío.

### PAdES

PDF Advanced Electronic Signatures, estándar europeo para firma digital
de PDFs. Es el formato de firma del PDF del testimonio, que es el documento
con valor legal.

### Hash SHA-256

Función criptográfica que produce un identificador único de 256 bits (64
caracteres hexadecimales) para cualquier archivo. Se usa para garantizar que
el PDF que llega al RPI es exactamente el que el escribano firmó.

### Callback / Webhook

Endpoint HTTP que un sistema expone para recibir notificaciones de otro
sistema. En este contrato, el sistema del Colegio expone un callback que el
RPI invoca para notificar cambios de estado del testimonio.

### multipart/form-data

Formato HTTP para enviar varias partes (campos, archivos) en una sola
petición. Se usa para enviar el XML y el PDF juntos.

### Backoff exponencial

Estrategia de reintentos donde el tiempo de espera entre intentos crece
exponencialmente (1s, 2s, 4s, 8s...). Reduce la carga sobre el servidor
cuando hay fallas transitorias.

## Recursos adicionales

Si querés profundizar:

- **Ley 17.801** — Régimen registral inmobiliario nacional argentino.
- **Ley provincial 1.946** de Neuquén — Régimen registral provincial.
- **W3C XML Signature Syntax and Processing** — https://www.w3.org/TR/xmldsig-core/
- **RFC 7515** — JSON Web Signature (para referencia, aunque no se usa en este contrato).

---

[← Índice de la documentación](README.md)