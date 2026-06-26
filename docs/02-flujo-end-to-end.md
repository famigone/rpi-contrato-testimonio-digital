# 02 — Flujo end-to-end

## Diagrama de secuencia

```
┌─────────────┐     ┌──────────────┐     ┌────────────┐     ┌─────────────┐
│  Escribano  │     │   Sistema    │     │  Servicio  │     │  Sistema    │
│             │     │  del Colegio │     │   del RPI  │     │  Registral  │
└──────┬──────┘     └──────┬───────┘     └─────┬──────┘     └──────┬──────┘
       │                   │                   │                   │
       │ Confecciona y     │                   │                   │
       │ firma testimonio  │                   │                   │
       ├──────────────────>│                   │                   │
       │                   │                   │                   │
       │                   │ POST /testimonios │                   │
       │                   │ (XML+PDF firmados)│                   │
       │                   ├──────────────────>│                   │
       │                   │                   │ Valida XSD        │
       │                   │                   │ Valida firma      │
       │                   │                   │ Verifica hash PDF │
       │                   │                   │ Persiste          │
       │                   │ 202 Accepted      │                   │
       │                   │ {idEnvio: UUID}   │                   │
       │                   │<──────────────────┤                   │
       │                   │                   │                   │
       │                   │                   │ Sincroniza datos  │
       │                   │                   ├──────────────────>│
       │                   │                   │                   │
       │                   │                   │                   │ Inscribe
       │                   │                   │                   │ provisorio
       │                   │                   │                   │ o definitivo
       │                   │                   │ POST callback     │
       │                   │                   │<──────────────────┤
       │                   │                   │                   │
       │                   │ POST /callback    │                   │
       │                   │ (estado, EG, ...) │                   │
       │                   │<──────────────────┤                   │
       │                   │ 200 OK            │                   │
       │                   ├──────────────────>│                   │
       │                   │                   │                   │
       │ Ve resultado al   │                   │                   │
       │ entrar al sistema │                   │                   │
       │<──────────────────┤                   │                   │
       │                   │                   │                   │
```

## Descripción paso a paso

### 1. El escribano confecciona el testimonio

El escribano, logueado en el sistema del Colegio, completa los datos de **uno o
más actos** (compraventas) que se realizarán: por cada acto, sus partes,
inmuebles, monto, certificaciones y visado; y a nivel testimonio, el
otorgamiento, el rogante y el cuerpo de la escritura. El sistema del Colegio
asiste en el armado de la escritura y genera:

- Un **XML** estructurado con los datos del testimonio, validable contra el XSD
  del contrato.
- Un **PDF** del testimonio completo (con todo el cuerpo de la escritura,
  cláusulas, identificación de partes, etc.).

### 2. El escribano firma digitalmente

El escribano firma **el XML** con su certificado digital (XML-DSig). La firma
del PDF puede hacerse aparte siguiendo los formatos PDF estándares de firma
digital (PAdES o similar) — el RPI verifica que el PDF esté firmado, no la
firma específica del XML sobre el PDF.

Detalles de la firma del XML están en [05 — Firma digital](05-firma-digital.md).

### 3. El sistema del Colegio envía el testimonio al RPI

El sistema del Colegio hace un `POST` al endpoint del RPI con:

- El XML firmado.
- El PDF firmado.

Formato: `multipart/form-data`. Detalles en [03 — Endpoint API](03-endpoint-api.md).

### 4. El RPI valida y persiste

El servicio del RPI:

1. Valida que el XML sea sintácticamente correcto y cumpla el XSD.
2. Valida la firma XML-DSig: que sea verificable, que el certificado del
   firmante sea válido, no esté revocado, etc.
3. Calcula el hash SHA-256 del PDF recibido y lo compara con el hash declarado
   en el XML (`MetadatosEnvio/HashPDF`).
4. Verifica idempotencia por `MetadatosEnvio/IdentificadorEnvio`: si ya recibió
   un testimonio con ese UUID, devuelve el resultado anterior.
5. Persiste el testimonio.
6. Almacena el PDF en el sistema de archivos seguro del RPI.
7. Responde **HTTP 202 Accepted** con el `identificadorEnvio` y estado inicial.

### 5. El RPI sincroniza con su sistema registral

De manera asincrónica (segundos a minutos después), el servicio del RPI
sincroniza los datos del testimonio con el sistema registral. El testimonio
entra a la cola de procesamiento del RPI con prioridad equivalente a una
minuta presentada en papel.

### 6. El sistema registral procesa el trámite

El calificador del RPI evalúa el testimonio. El resultado posible es:

- **Inscripción definitiva**: el trámite se inscribe.
- **Inscripción provisoria** con VIP (Vista de Insubsanable Pendiente o
  Volante de Inscripción Provisoria): el trámite tiene observaciones que deben
  subsanarse.
- **Rechazo**: el trámite no se inscribe (causales graves).

### 7. El RPI notifica al Colegio el resultado

Cuando hay un evento relevante en el ciclo de vida del testimonio (inscripción
provisoria, definitiva, rechazo, etc.), el RPI hace un `POST` al endpoint que
el Colegio expone para recibir callbacks.

El sistema del Colegio actualiza su base interna y muestra al escribano el
resultado la próxima vez que se loguee.

Detalles en [08 — Notificaciones de callback](08-notificaciones-callback.md).

## Tiempos esperados

Para que el sistema del Colegio pueda diseñar la experiencia del escribano:

| Evento | Tiempo típico esperado |
|--------|------------------------|
| Respuesta inicial al POST (202 Accepted) | < 5 segundos |
| Inscripción definitiva o provisoria | varía según carga del RPI (minutos a horas hábiles) |
| Notificación al callback del Colegio | segundos después del evento registral |

Estos tiempos son orientativos y pueden variar.

## Idempotencia

El sistema del Colegio puede reenviar el mismo testimonio (mismo
`identificadorEnvio`) sin riesgo de duplicación. El RPI detecta el reenvío
y devuelve el resultado del envío original.

Esto permite manejar errores de red, timeouts y reintentos sin lógica
adicional del lado del Colegio.

## Próximos pasos

- Para el contrato técnico del endpoint, seguí con
  [03 — Endpoint API](03-endpoint-api.md).
- Para el formato del XML, andá a [04 — Formato XML](04-formato-xml.md).
