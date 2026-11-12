# Política de seguridad

Este documento describe cómo reportar vulnerabilidades o problemas de
seguridad relacionados con el contrato del Testimonio Digital.

---

## ⚠️ Importante

**No reportes vulnerabilidades en issues públicos.** Si descubrís una
debilidad de seguridad en este contrato o en cómo se procesa, seguí el
proceso descrito abajo.

---

## Qué se considera un problema de seguridad

Ejemplos de cosas que califican:

- Una vulnerabilidad en la verificación de firma digital descrita en este
  contrato (por ejemplo, una manera de pasar la validación con un
  certificado inválido).
- Un caso donde la idempotencia descrita puede ser eludida para inyectar
  testimonios duplicados.
- Una manera de hacer que el servicio del RPI procese contenido malicioso
  embebido en el XML o el PDF.
- Una ambigüedad en el contrato que permita interpretaciones contradictorias
  que comprometen la seguridad (ej. dos clientes interpretan distinto el
  campo de hash y eso permite ataques).
- Una debilidad criptográfica en los algoritmos elegidos (ej. si SHA-256 se
  considera comprometido en el futuro).
- Una manera de que un canal lea **novedades, testimonios o documentos de
  otro**. El alcance de la consulta está atado al canal con el que se envió
  cada testimonio, no al token concreto, y es la propiedad que sostiene todo
  el mecanismo: la lista de un token trae únicamente las novedades de sus
  propios envíos.
- Una manera de **confirmar la existencia** de un recurso ajeno. Una
  `referencia` de documento o un `identificadorEnvio` de otro canal se
  contestan con **404**, igual que uno inexistente, y a propósito: un 403
  confirmaría que existe. Un camino que distinga los dos casos —por el código,
  por el cuerpo, por el tiempo de respuesta— es un problema de seguridad.
- Una manera de que un acuse marque como recibidas novedades que **no se
  entregaron**, o de que una novedad se pierda sin acusar. El acuse enumera
  secuencias justamente para eso ([08](docs/08-novedades.md)).
- Un problema en el registro de descargas que permita **negar o falsear** una
  descarga que ocurrió: de ese registro depende el momento con efectos
  legales.

---

## Qué NO califica acá

- Bugs en el servicio del RPI que no son del contrato en sí (eso va al
  repositorio del servicio).
- Bugs en sistemas integradores (eso va al repositorio del integrador).
- Problemas funcionales que no afectan la seguridad (eso es un issue
  normal en este repo).
- Quejas sobre rendimiento o latencia.

---

## Cómo reportar

> ⚠️ **Información de contacto pendiente de definición**. El equipo
> técnico del RPI establecerá un canal específico para reportes de
> seguridad antes de la primera publicación oficial del contrato.

Mientras tanto, los reportes de seguridad deben enviarse a:

- **Email**: `[POR-DEFINIR]@jusneuquen.gov.ar`
- **Alternativa**: contacto directo al líder técnico del RPI Neuquén.

En el reporte incluí:

1. **Descripción del problema**: qué es, en qué versión del contrato aplica.
2. **Impacto**: qué se puede hacer aprovechando el problema.
3. **Reproducción**: pasos para reproducirlo.
4. **Sugerencia de mitigación** (si tenés alguna idea).
5. **Datos de contacto** para que el RPI pueda responderte.

---

## Qué esperar después del reporte

1. **Acuse de recibo**: dentro de las 72 horas hábiles.
2. **Análisis inicial**: el equipo evalúa el reporte y confirma si es
   un problema válido. Plazo orientativo: 7 días hábiles.
3. **Comunicación del plan**: si es válido, el equipo informa qué hará
   y un cronograma estimado.
4. **Mitigación**: dependiendo de la gravedad y complejidad del problema.
5. **Disclosure coordinado**: cuando el problema esté resuelto, se
   comunica públicamente el problema y la solución, dándole crédito al
   reportante si lo desea.

---

## Compromiso

El equipo del RPI se compromete a:

- Tomar en serio cada reporte.
- Responder en plazos razonables.
- No tomar represalias contra reportantes que actúen de buena fe.
- Dar crédito público al reportante (con su consentimiento).

---

## Vulnerabilidades conocidas

Las vulnerabilidades históricas, una vez resueltas y comunicadas
públicamente, se listan en:

- [CHANGELOG.md](CHANGELOG.md) bajo la sección de la versión que las corrigió.

Si tenés acceso al historial de releases, podés ver allí las
vulnerabilidades de seguridad ya resueltas.

---

## Algoritmos criptográficos

Los algoritmos definidos en este contrato (RSA-SHA256, Canonical XML 1.0,
SHA-256 para hash de PDF) reflejan el estado del arte al momento de
publicación de la versión.

Si en el futuro alguno de estos algoritmos se considera comprometido, el
equipo del RPI publicará una versión MAJOR del contrato con algoritmos
actualizados, siguiendo el proceso definido en
[GOVERNANCE.md](GOVERNANCE.md).
