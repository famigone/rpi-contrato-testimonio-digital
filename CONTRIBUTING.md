# Cómo contribuir

Este documento describe cómo proponer cambios al contrato del Testimonio
Digital del RPI Neuquén. Si querés entender el contexto más amplio (roles,
versionado, políticas), leé primero [GOVERNANCE.md](GOVERNANCE.md).

---

## ¿Querés hacer una pregunta?

**Abrí un issue** con la etiqueta `pregunta`. No tenés que abrir un PR para
hacer una pregunta.

Buenas preguntas son:

- "El XSD permite N adquirentes con proporción 1/2 cada uno pero suma > 1.
  ¿Está bien así o debería validarse en el cliente?"
- "Si el escribano firma con un certificado emitido por Y autoridad
  certificante, ¿es aceptado?"
- "El campo X dice 'opcional' pero el ejemplo no lo incluye. ¿Es realmente
  opcional o el ejemplo es incompleto?"

---

## ¿Encontraste un error?

Issue con etiqueta `bug` o `documentación`. Incluí en el issue:

- Versión del contrato afectada.
- Descripción del problema.
- Si aplica, qué documento o XSD tiene el error.
- Si es un problema con un ejemplo, qué pasa al validarlo.

---

## ¿Querés proponer un cambio?

**Antes de abrir un PR**, abrí un issue de propuesta. Esto evita trabajo
desperdiciado si la propuesta no encaja con la dirección del contrato.

El issue debe incluir:

1. **Problema o necesidad** que motivó la propuesta.
2. **Propuesta concreta** del cambio.
3. **Impacto** en consumidores existentes (retro-compatible o no).
4. **Alternativas** consideradas y por qué se descartan.

El mantenedor responde indicando si la propuesta tiene sentido para abrir un PR.

---

## Tipos de cambios

| Tipo | Ejemplo | Versión que dispara |
|------|---------|---------------------|
| Correcciones de typos | Errores en docs | Patch |
| Mejoras en docs | Más detalle, más ejemplos | Patch |
| Ejemplos nuevos | Caso de uso adicional | Patch |
| Aclaración de un campo existente | Documentación clarificada | Patch |
| Agregar campo opcional al XML | Nuevo campo no obligatorio | Minor |
| Agregar tipo de acto | Hipoteca, donación, etc. | Minor |
| Agregar código de error nuevo | Cobertura de caso no manejado | Minor |
| Cambiar el namespace XML | Migración mayor | Major |
| Eliminar un campo del XML | Rompe clientes | Major |
| Cambiar mecanismo de autenticación | Cambio crítico | Major |

Ver [GOVERNANCE.md](GOVERNANCE.md) para los criterios completos.

---

## Cómo abrir un PR

### Convenciones de branch

- Para correcciones: `fix/descripcion-corta`
- Para agregados: `feat/descripcion-corta`
- Para docs: `docs/descripcion-corta`
- Para cambios breaking: `breaking/descripcion-corta`

### Antes del PR

1. **Validar XSDs**: si modificaste algún XSD, validar que los ejemplos
   siguen pasando:

   ```bash
   xmllint --schema xsd/testimonio-digital.xsd ejemplos/compraventa-minima.xml --noout
   xmllint --schema xsd/testimonio-digital.xsd ejemplos/compraventa-multiple-titulares.xml --noout
   xmllint --schema xsd/testimonio-digital.xsd ejemplos/compraventa-usd.xml --noout
   ```

2. **Verificar links de markdown**: links internos a otros documentos no
   deben estar rotos.

3. **Actualizar CHANGELOG.md**: agregar entrada bajo `[No publicado]`
   describiendo el cambio.

4. **Si agregaste un campo nuevo**: agregar al menos un ejemplo que lo
   use.

5. **Si cambiaste un XSD**: revisar que la docs en `docs/04-formato-xml.md`
   refleje el cambio.

### Estructura del PR

Título: descriptivo, con prefijo del tipo de cambio.

```
feat: agregar tipo de acto Hipoteca
fix: corregir longitud máxima del campo Nombre
docs: aclarar política de reintentos en errores 5xx
breaking: cambiar formato de Proporción a decimal
```

Cuerpo del PR debe responder:

1. **Qué cambió** (resumen).
2. **Por qué** (link al issue de propuesta).
3. **Impacto en consumidores** (retro-compatible o no).
4. **Cómo se probó** (XSDs validados, links revisados).
5. **Checklist**.

Usá este template:

```markdown
## Qué cambió

[Resumen]

## Por qué

Cierra #[número del issue de propuesta].

[Motivo del cambio]

## Impacto en consumidores

[ ] Retro-compatible: los clientes existentes no necesitan cambios.
[ ] Retro-compatible con depreciación: los clientes existentes funcionan
     pero hay una nueva forma recomendada.
[ ] Breaking: los clientes existentes deben actualizarse.

## Cómo se probó

[ ] Los 3 ejemplos validan contra el XSD modificado.
[ ] Los links internos de markdown no están rotos.
[ ] El CHANGELOG fue actualizado.
[ ] Si es breaking, hay guía de migración.

## Notas adicionales

[Lo que el revisor debería saber para evaluar el PR]
```

---

## Proceso de revisión

1. **Revisor automático/básico**: validación de XSD, links.
2. **Revisión del mantenedor**: alineación con la dirección del contrato,
   coherencia, calidad del cambio.
3. **Revisión adicional** (si aplica):
   - Cambios funcionales: revisor del lado funcional (Dirección de
     Notariado).
   - Cambios de seguridad: revisor del Poder Judicial.
   - Cambios breaking: revisor del integrador principal.
4. **Aprobación y merge**: el mantenedor mergea cuando todas las
   aprobaciones están.

---

## Style guide

### Markdown

- Headers con `#`, no con `===` o `---`.
- Lineas de 80-100 caracteres cuando sea posible.
- Listas con `-` (no `*`).
- Bloques de código con triple backtick e indicación de lenguaje cuando
  aplica.
- Tablas para datos estructurados (códigos de error, campos, etc.).
- Links a otros documentos del repo con paths relativos.

### XSD

- Usar `xs:annotation/xs:documentation` para todo tipo o elemento principal.
- Documentación en español.
- Nombres de tipos: `PascalCase` terminados en `Type` (ej. `PersonaType`).
- Nombres de elementos: `PascalCase` (ej. `Adquirentes`).
- Nombres de atributos: `camelCase` (ej. `version`).
- Restricciones de longitud y patrón sobre todos los strings que viajen al
  RPI.

### Ejemplos XML

- Datos ficticios (CUITs, DNIs, matrículas).
- Los ejemplos deben validar contra el XSD.
- Si un ejemplo nuevo usa un campo opcional, documentar al inicio qué
  representa el caso.

---

## Qué NO va en este repo

- Configuración interna del servicio del RPI (env vars, deploy, scripts).
- URLs concretas de IPs internas del Poder Judicial.
- Credenciales, secrets, tokens.
- Información que pueda comprometer la seguridad del servicio.

Esa información vive en el repositorio del servicio (`rpi-td`), no
acá.

---

## Preguntas frecuentes

### ¿Puedo abrir un PR sin issue previo?

Para cambios de patch (typos, mejoras menores de documentación), sí. Para
cambios minor y major, abrí primero el issue.

### ¿Cuánto tarda en revisarse un PR?

Patch: pocos días. Minor: 1-2 semanas (incluye período de discusión).
Major: mínimo 30 días por el período de discusión pública.

### ¿Puedo proponer un cambio breaking?

Sí, pero implica un proceso más largo. Ver [GOVERNANCE.md](GOVERNANCE.md)
sección "Major (`X.0.0`)".

### ¿Qué pasa si mi PR es rechazado?

El mantenedor explica el motivo en el PR. Podés:

- Discutir y proponer ajustes.
- Cerrar el PR y abrir uno nuevo con un enfoque distinto.
- Si tenés un desacuerdo profundo, ver "Resolución de conflictos" en
  [GOVERNANCE.md](GOVERNANCE.md).

---

## Gracias

Cada contribución mejora el contrato. Los issues, PRs, preguntas y
sugerencias suman al ecosistema.
