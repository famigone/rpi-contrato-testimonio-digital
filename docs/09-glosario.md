# 09 — Glosario

Términos del dominio registral y notarial que aparecen en este contrato.

## Términos generales

### Acto

La operación jurídica que se inscribe en el RPI. En v2.0 del contrato, el único
tipo soportado es la **compraventa**, pero un mismo testimonio puede contener
**N actos** (elemento `<Acto>` dentro de `<Actos>`). Otros tipos de acto son
hipoteca, donación, división de condominio, inhibición, etc.

### Adquirente

Persona que **recibe** el inmueble en una compraventa. En el sistema legacy
se llama "titular adquirente". En el contrato v2 es una `<Parte rol="ADQUIRENTE">`.

### Parte

Persona que interviene en un acto. En v2 cada acto tiene una lista de `<Parte>`,
donde el rol se indica con el atributo `rol`. Reemplaza los contenedores
`Adquirentes` / `Transmitentes` de v1. Una `Parte` es una persona (con su `Tipo`
H/J/O, representante y proporción) más el atributo `rol`.

### Rol de parte

Atributo obligatorio de cada `<Parte>` que indica su papel en el acto. Roles
definidos: `ADQUIRENTE`, `TRANSMITENTE`, `ACREEDOR`, `DEUDOR`
(`ACREEDOR`/`DEUDOR` quedan listos para hipoteca, aún no disponible). El XSD
acepta cualquier rol en cualquier acto; la correspondencia rol/tipo-de-acto la
valida el servicio del RPI.

### Asiento

Cada anotación que se hace en una matrícula del registro. Cada vez que se
inscribe un acto sobre un inmueble, se registra un asiento nuevo.

### Calificador

Funcionario del RPI que evalúa si un trámite cumple los requisitos para ser
inscripto. Puede inscribir, observar (provisoriamente) o rechazar.

### Certificación registral

Certificado emitido por el RPI antes del acto, que informa el estado dominial
del inmueble. El escribano lo solicita antes de otorgar la escritura para
asegurarse del estado del bien.

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
puede tener observaciones.

### EG / Entrada General

Número que el RPI asigna a cada trámite cuando ingresa. Es el identificador
único del trámite dentro del RPI. Tiene un número y un año (ej: 12345/2026).

Hoy, en el flujo en papel, la EG se asigna cuando el operador del RPI escanea
el código de barras o ingresa manualmente la minuta. En el flujo de testimonio
digital, la asigna el proceso interno del RPI cuando el testimonio entra a la
cola.

### Inscripción definitiva

Estado final del trámite cuando todo está conforme y queda anotado en la
matrícula del inmueble. Tiene plena eficacia legal.

### Inscripción provisoria

Estado del trámite cuando hay observaciones que el escribano debe subsanar
antes de la inscripción definitiva. El trámite se anota provisoriamente
mientras se subsanan los defectos. Tiene una vigencia limitada (típicamente
60 días) durante la cual se mantiene el orden de prelación.

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
de longitud fija (2, 2, 3, 4 y 4 caracteres) según la convención del RPI. Es un
bloque opcional.

### Plataforma Digital (PD)

Canal digital existente del RPI para formularios estructurados (Dominio,
Inhibición, Afectación Vivienda). Convive con el canal del testimonio digital.

### Mesa de Entradas Digital (MED)

Canal actual del RPI para presentación de testimonios en papel con
acompañamiento de minuta digital. Convive con el canal del testimonio digital.

### Rechazo registral

Cuando el calificador determina que el trámite tiene defectos graves que no
pueden subsanarse (por ejemplo, matrícula que no corresponde) y rechaza la
inscripción.

### Rogante

Persona que actúa como **representante del trámite ante el RPI**. Es quien
"ruega" la inscripción. Típicamente es el escribano autorizante, pero puede
ser otra persona designada. El RPI se comunica con el rogante para
notificaciones formales del trámite.

### Tomo / Folio / Finca

Sistema de identificación de inmuebles previo al folio real (matrícula). Se usa
para inmuebles antiguos que aún no fueron matriculados. En el contrato son tres
campos opcionales que se completan en conjunto cuando el inmueble no tiene
matrícula.

### Tasa registral

Pago al RPI por el servicio de inscripción del trámite. Se acredita mediante
un número de tasa que el escribano gestiona antes del envío.

### Testimonio

Copia autenticada por el escribano de la escritura matriz, con valor de título
inscribible. Es lo que históricamente se presenta en papel al RPI.

### Transmitente

Persona que **entrega** el inmueble en una compraventa. En el sistema legacy
se llama "titular transmitente". En el contrato v2 es una
`<Parte rol="TRANSMITENTE">`.

### Visado de Rentas

Validación previa al acto por la Dirección Provincial de Rentas. En el contrato
es un bloque obligatorio: con visado se indica `Tipo=R` (y se incluye el número
de trámite); sin visado (exento u otra causal) se indica `Tipo=A`.

### VIP / VIO / Volante de Inscripción Provisoria

Documento que el RPI emite cuando un trámite queda inscripto provisoriamente,
detallando las observaciones que deben subsanarse. Por extensión, se usa "VIP"
para referirse al estado de inscripción provisoria en sí.

## Términos notariales

### Asentimiento conyugal

Cuando el inmueble es ganancial (durante el matrimonio en régimen de comunidad),
el cónyuge del transmitente debe prestar asentimiento para la venta. En v2.0
del contrato esto se modela como **texto libre** en el campo
`AsentimientoConyugal` dentro de `Compraventa`, no como bloque estructurado.

### Escribano autorizante

El notario que firma la escritura y da fe del acto. Es quien firma
digitalmente el XML del testimonio digital.

### Escritura

Documento notarial original (matriz) que queda en el protocolo del escribano.
El testimonio es una copia autenticada de la escritura.

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

En una compraventa con varios adquirentes, indica qué fracción del inmueble
adquiere cada uno. Se modela como string fracción (`"1/2"`, `"1/3"`, `"1/1"`)
dentro de la `<Parte>`. La validación de que las proporciones sumen 1 **por
acto** la hace el servicio del RPI.

### Representante

Persona que actúa por cuenta de otra (tutor, apoderado, etc.). En el contrato
es un bloque opcional dentro de cada `<Parte>`, cualquiera sea su rol.

### Protocolo

Libro encuadernado donde el escribano archiva las escrituras matrices.

## Términos técnicos del contrato

### Idempotencia

Propiedad de una operación que produce el mismo resultado sin importar cuántas
veces se ejecute. En este contrato, enviar el mismo testimonio (mismo
`IdentificadorEnvio`) varias veces produce siempre el mismo resultado: el
testimonio se procesa una sola vez.

### IdentificadorEnvio

UUID v4 generado por el sistema del Colegio para identificar unívocamente
cada envío de testimonio. Si hay que reintentar por error de red, se reusa el
mismo UUID.

### XML-DSig

XML Digital Signature, estándar W3C para firma digital de documentos XML.
El XML del testimonio se firma con XML-DSig por el escribano autorizante.

### PAdES

PDF Advanced Electronic Signatures, estándar europeo para firma digital
de PDFs. Recomendado para firmar el PDF del testimonio.

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
