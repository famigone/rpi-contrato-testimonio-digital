# Gobernanza

Este documento describe **cómo se mantiene y evoluciona** el contrato del
Testimonio Digital del RPI Neuquén. Define roles, procesos de cambio y
políticas que aplican a este repositorio.

---

## Principios

1. **El contrato es un activo compartido**. Lo usa el RPI internamente, los
   sistemas notariales que integran (en v1.0: el Colegio de Escribanos de
   Neuquén), y eventualmente otros consumidores.

2. **Los cambios incompatibles tienen costo institucional**. Romper un contrato
   en producción significa que clientes existentes dejan de funcionar. Por eso
   los cambios breaking requieren proceso explícito.

3. **La estabilidad es valiosa**. Cambiar el contrato cada mes erosiona la
   confianza de los integradores. Los cambios deben tener motivo claro.

4. **La transparencia mejora la calidad**. Las decisiones se documentan, las
   discusiones quedan en issues/PRs, los cambios se justifican en commits y
   CHANGELOG.

---

## Roles

### Mantenedor

El **mantenedor** del contrato es el responsable final de las decisiones de
cambio. En esta etapa del proyecto, el mantenedor es el **Equipo Técnico del
RPI Neuquén**.

Responsabilidades del mantenedor:

- Aprobar o rechazar cambios al contrato.
- Coordinar releases.
- Comunicar cambios breaking a los consumidores conocidos.
- Mantener actualizado el CHANGELOG.
- Garantizar la coherencia entre la documentación y la implementación.

### Revisores

Personas que pueden revisar propuestas de cambio sin ser mantenedoras. Su
revisión no es vinculante pero ayuda al mantenedor a tomar mejores decisiones.

En v1.0 incluyen al menos:

- Responsable técnico del Colegio de Escribanos de Neuquén (para cambios que
  afectan al integrador principal).
- Dirección de Notariado (para cambios funcionales que requieran validación
  normativa).
- Equipo de Infraestructura y Seguridad del Poder Judicial (para cambios de
  autenticación, firma o aspectos de seguridad).

### Contribuyentes

Cualquier persona que abra un issue o un PR con una propuesta. No requiere
permisos especiales.

---

## Tipos de cambios

### Patch (`1.0.X`)

Cambios que **no modifican el contrato técnico**:

- Correcciones de typos.
- Aclaraciones en la documentación.
- Ejemplos nuevos o mejorados.
- Mejoras en la organización del repositorio.
- Documentación interna (gobernanza, contribución, etc.).

**Proceso**: PR con revisión del mantenedor. Aprobación de un revisor opcional.

### Minor (`1.X.0`)

Cambios que **agregan funcionalidad de manera retro-compatible**:

- Agregar un nuevo tipo de acto (ej. hipoteca en `xsd/actos/hipoteca.xsd`).
- Agregar un campo opcional al XML.
- Ampliar el rango de valores aceptados en un campo existente.
- Agregar un nuevo código de error.
- Agregar un nuevo tipo de evento de callback.

**Criterio de retro-compatibilidad**: un cliente que cumplía con la versión
anterior debe seguir funcionando sin cambios contra la versión nueva.

**Proceso**:

1. Issue describiendo la propuesta y su motivación.
2. Discusión pública (mínimo 7 días para permitir revisión de los consumidores).
3. PR con los cambios + actualización de CHANGELOG.
4. Revisión del mantenedor + al menos un revisor.
5. Merge y tag de release.

### Major (`X.0.0`)

Cambios **incompatibles** con versiones anteriores:

- Eliminar un campo del XML.
- Cambiar el tipo o formato de un campo existente de manera incompatible.
- Cambiar la estructura general del XML.
- Cambiar el mecanismo de autenticación de manera incompatible.
- Cambiar el formato de los callbacks.
- Cambiar el namespace XML.

**Proceso especial para cambios MAJOR**:

1. **Issue de propuesta** con justificación detallada del motivo del cambio.
2. **Período de discusión pública**: mínimo 30 días.
3. **Aviso formal a consumidores conocidos**: el mantenedor notifica
   explícitamente a cada consumidor identificado.
4. **PR con los cambios** + actualización completa de CHANGELOG + guía de
   migración (`docs/migracion-vX-a-vY.md`).
5. **Aprobación de mantenedor + al menos dos revisores**, incluyendo
   obligatoriamente:
   - Revisor del lado del integrador principal (en v1: Colegio).
   - Revisor del lado del RPI (Dirección de Notariado o Líder Técnico).
6. **Período de coexistencia**: la versión anterior y la nueva conviven en
   producción durante un período definido (sugerido: 6 meses) antes de
   deprecar la anterior.

---

## Período de coexistencia

Cuando se publica una versión MAJOR nueva, **el RPI mantiene operativos
simultáneamente**:

- La versión anterior (en su namespace antiguo).
- La versión nueva (en el namespace nuevo).

Los clientes pueden migrar a su ritmo durante el período de coexistencia. Al
finalizar el período:

- La versión anterior se marca como **deprecada**.
- Los envíos contra la versión deprecada generan una advertencia (en el
  callback o en headers de respuesta).
- Después de un período adicional de deprecación, se desactiva.

El cronograma completo (publicación → deprecación → desactivación) se anuncia
con la publicación de la versión nueva.

---

## Proceso de release

### Versión patch

1. Mergear los PRs pendientes a la rama principal.
2. Actualizar `CHANGELOG.md` agregando la nueva versión.
3. Crear tag Git: `git tag -a v1.0.1 -m "Release v1.0.1"`.
4. Hacer push del tag.
5. Crear Release en la plataforma Git con notas de release.

### Versión minor

Idem patch, pero:

- El CHANGELOG describe los agregados con más detalle.
- La nota de release menciona explícitamente que es retro-compatible.

### Versión major

Idem minor, pero:

- Incluye guía de migración.
- Anuncia el cronograma de deprecación de la versión anterior.
- Notifica explícitamente a los consumidores conocidos antes del release.

---

## Criterios para considerar un cambio

Antes de aprobar un cambio, el mantenedor evalúa:

| Criterio | Pregunta |
|----------|----------|
| Necesidad | ¿Qué problema concreto resuelve? |
| Alternativas | ¿Hay otras maneras de resolverlo sin cambiar el contrato? |
| Impacto | ¿Qué consumidores se ven afectados? |
| Costo | ¿Cuál es el costo de implementación + migración? |
| Retro-compatibilidad | ¿Se puede hacer compatible? |
| Coherencia | ¿Es consistente con el resto del contrato? |
| Implementabilidad | ¿Se puede implementar del lado del RPI? |
| Reversibilidad | Si descubrimos que fue una mala idea, ¿se puede revertir? |

---

## Decisiones que requieren acuerdo bilateral

Algunos cambios afectan directamente al integrador principal y requieren
acuerdo explícito antes de mergear:

- Cambio de URL del endpoint en producción.
- Cambio del mecanismo de autenticación.
- Cambio del formato o autenticación de los callbacks.
- Deprecación de campos que el integrador está usando.

Para estos casos, el PR no se mergea sin aprobación documentada del
integrador (puede ser un comentario formal en el PR o un mail referenciado
en el PR).

---

## Resolución de conflictos

Si hay desacuerdo entre el RPI y un integrador sobre un cambio propuesto:

1. **Discusión técnica**: documentar las posiciones en el issue/PR.
2. **Escalamiento al nivel directivo**: si no hay acuerdo técnico, escalan
   los responsables institucionales (Dirección del RPI y autoridad
   competente del integrador).
3. **El RPI tiene la decisión final** como dueño del contrato, pero ese
   poder se ejerce con prudencia y considerando el impacto en el ecosistema.

---

## Comunicación

### Canales

- **Issues**: para reportar problemas, hacer preguntas, proponer cambios.
- **Pull Requests**: para implementar cambios propuestos.
- **Releases**: para anuncios de versión nueva.
- **CHANGELOG.md**: historial completo navegable.

### Avisos a consumidores

Cuando se publica una versión:

- **Patch**: solo actualización del CHANGELOG. No requiere aviso individual.
- **Minor**: aviso vía Release Notes. Se notifica a la lista de consumidores
  conocidos (mailing list o canal acordado).
- **Major**: aviso formal individual a cada consumidor identificado, con
  fecha estimada de publicación, fecha de inicio de período de coexistencia
  y fecha de deprecación.

---

## Documentos relacionados

- [CONTRIBUTING.md](CONTRIBUTING.md) — cómo contribuir prácticamente.
- [CHANGELOG.md](CHANGELOG.md) — historial de versiones.
- [SECURITY.md](SECURITY.md) — reporte de problemas de seguridad.
