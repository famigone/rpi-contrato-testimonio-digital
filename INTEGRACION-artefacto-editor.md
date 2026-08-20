# Guía de integración — Artefacto de campos ↔ Editor del escribano

> **Para:** programadores del editor del Colegio de Escribanos.
> **Qué es esto:** cómo consumir `artefacto-campos-por-acto.json` para armar dinámicamente el
> formulario de carga de un testimonio digital, validarlo, y enviarlo al RPI.
> **Versión del artefacto que documenta:** esquema `1.0.0` / contenido `1.0.0`.

---

## 1. Panorama: las tres piezas del repo del contrato

El editor consume tres archivos, con roles distintos. **Ninguno reemplaza a los otros.**

| Archivo | Qué es | Para qué lo usa el editor |
|---------|--------|---------------------------|
| `catalogo-actos.json` | Los 231 actos con su nombre | Armar el **selector de actos** |
| `artefacto-campos-por-acto.json` | Reglas de campos de los actos habilitados | Armar el **formulario** de cada acto |
| XSD del contrato (v3) | Estructura del XML del testimonio | **Validar** el XML antes de enviarlo |

Flujo mental: el **catálogo** dice qué actos ofrecer, el **artefacto** dice qué campos pedir por
acto, el **XSD** dice si el XML resultante es válido.

> **Importante — actos mostrados vs habilitados.** El catálogo tiene 231 actos; el artefacto solo
> los **habilitados** (hoy 59). Un acto que está en el catálogo pero **no** en el artefacto **existe
> pero todavía no se puede enviar por testimonio digital**. Recomendación de UX: mostrar los 231 en
> el selector, pero **deshabilitar** (grisar) los que no están en el artefacto, con un aviso tipo
> "no disponible por testimonio digital aún". No los ocultes: el escribano tiene que ver que el acto
> existe y que está en camino.

---

## 2. El artefacto tiene dos niveles

El JSON se divide en `esquema` (nivel 1) y `actos` (nivel 2). **Cambian a ritmos distintos, y esa
separación es el contrato de integración.**

### Nivel 1 — `esquema` (clave `versionEsquema`)
Define **cómo se describe** cualquier acto: las estructuras genéricas. Cambia rara vez.
Contiene:
- `tiposDePersona` (`H`/`J`/`O`) — qué campos tiene cada tipo de persona.
- `camposCondicionalesPersona` — campos extra que aparecen bajo ciertas condiciones.
- `tiposDeCampo` — la forma de un `monto`, un `certificado`, un `inmueble`.
- `roles` — los tres roles genéricos y su etiqueta de UI.
- `labelsCampo` — los textos de los campos, tal como los ve el escribano.
- `enums` — las listas cerradas para los desplegables.

### Nivel 2 — `actos` (clave `versionContenido`)
Por cada acto habilitado, **qué exige ese acto concreto**: qué partes, qué campos, con qué
obligatoriedad. Cambia cada vez que el RPI habilita un acto nuevo.

### Regla de versionado — LÉASE CON ATENCIÓN
- **`versionEsquema` es tu contrato.** Mientras no cambie, tu editor sigue funcionando **aunque el
  RPI habilite 100 actos más**. Un cambio de `versionEsquema` es *breaking*: probablemente tengas
  que tocar código (apareció un tipo de campo nuevo, una estructura nueva).
- **`versionContenido` cambia solo** cuando se habilitan/modifican actos. **No requiere cambios de
  código** — solo volver a descargar el artefacto. Tu editor debe tolerar que aparezcan actos nuevos
  en `actos` sin romperse.

**Cómo detectar una versión nueva:** compará `versionEsquema` / `versionContenido` del artefacto
que tenés cacheado contra el publicado. Si cambió `versionContenido`, refrescá los datos. Si cambió
`versionEsquema`, revisá el changelog antes de actualizar (puede requerir ajustes).

---

## 3. Estructura del Nivel 1 (referencia)

### `tiposDePersona`
Cada tipo (`H` humana, `J` jurídica, `O` organismo) trae un `label` y una lista `campos`:

```json
"H": {
  "label": "Persona humana",
  "campos": ["Tipo","ApellidoODenominacion","Nombres","CUIT","TipoDocumento","NumeroDocumento","PEP"]
}
```

Estos son los campos **base** de una persona de ese tipo. El editor renderiza un input por cada uno.
Los nombres coinciden con los elementos del XSD (así serializás directo).

### `camposCondicionalesPersona`
Campos que **se agregan** a una persona bajo una condición:

```json
"datosHumanosCompletos": ["EstadoCivil","Nacionalidad","FechaNacimiento"],
"asentimientoConyugal": ["Conyuge","Nupcias"]
```

- `datosHumanosCompletos`: se piden **solo si la parte lo marca** (ver `datosHumanosCompletos` en la
  parte, nivel 2). Aplica a personas humanas en ciertos roles.
- `asentimientoConyugal`: los campos del cónyuge que asiente, cuando corresponde (ver §5.1).

### `tiposDeCampo`
La forma de los campos no-persona:

```json
"monto":       { "campos": ["valor","moneda","cotizacion?"] },
"certificado": { "campos": ["numero","fechaEmision"] },
"inmueble":    { "campos": ["identificacion","nomenclaturaCatastral?","certCatastral?"] }
```
El sufijo `?` marca campo opcional.

### `roles`
```json
"wmt": { "label": "Adquirente / Titular" },
"wmc": { "label": "Transmitente" },
"wmi": { "label": "Acreedor" }
```
**Usá estas etiquetas tal cual.** Son las que el escribano ya conoce de la minuta actual. **No las
traduzcas por acto** (no pongas "Donante"/"Donatario" en una donación) — el sistema siempre rotula
genérico, y así lo espera el escribano.

### `labelsCampo` y `enums`
`labelsCampo` da el texto de cada campo de acto ("Precio o Monto de la Operación", etc.).
`enums` da las listas cerradas: `tipoDocumento`, `estadoCivil`, `moneda` (`["$","USD"]`),
`tipoPersona`, `tipoVisado`. Usalos para poblar los desplegables — no inventes opciones.

---

## 4. Estructura del Nivel 2 (una entrada de acto)

Ejemplo real — **1075 HIPOTECA**:

```json
"1075": {
  "nombre": "HIPOTECA",
  "familia": "HIPOTECARIA_CONSTITUYE",
  "partes": [
    { "rol": "wmi", "label": "Acreedor", "obligatorio": true, "cardinalidad": "1..n",
      "proporcion": { "admite": true, "sumaUno": true },
      "datosHumanosCompletos": false,
      "inhibicion": { "admite": false, "obligatoria": false },
      "asentimiento": false },
    { "rol": "wmt", "label": "Adquirente / Titular", "obligatorio": true, "cardinalidad": "1..n",
      "proporcion": { "admite": true, "sumaUno": false },
      "datosHumanosCompletos": true,
      "inhibicion": { "admite": true, "obligatoria": true },
      "asentimiento": true } ],
  "campos": {
    "precioOperacion": { "aplica": false, "obligatorio": false },
    "valuacionFiscal": { "aplica": false, "obligatorio": false },
    "montoHipoteca":   { "aplica": true,  "obligatorio": true },
    "certDominio":     { "aplica": true,  "obligatorio": true },
    "certCatastral":   { "aplica": true,  "obligatorio": true },
    "asentimiento":    { "aplica": true,  "rol": "wmt", "condicional": "estadoCivil" },
    "reconHipoteca":   { "aplica": true,  "obligatorio": false },
    "afectaciones":    { "aplica": true,  "obligatorio": false } }
}
```

### Semántica de cada atributo

**En `partes[]`:**
| Atributo | Significado | Qué hace el editor |
|----------|-------------|--------------------|
| `rol` | `wmt`/`wmc`/`wmi` | Identifica el grupo de partes; su etiqueta sale de `roles` (nivel 1) |
| `label` | Etiqueta ya resuelta | Podés usarla directo (es la de `roles`) |
| `obligatorio` | ¿Debe haber al menos una? | Si `true`, exigí ≥1 persona en ese rol |
| `cardinalidad` | `"1..n"` = una o varias | Permití agregar N personas al rol |
| `proporcion.admite` | ¿Esta parte lleva campo proporción? | Si `true`, mostrá input de proporción por persona |
| `proporcion.sumaUno` | ¿Las proporciones deben sumar 1? | Si `true`, **validá que sumen 1** (ver §5.2) |
| `datosHumanosCompletos` | ¿Pide estado civil / nacionalidad / fecha nac.? | Si `true`, agregá esos campos (de `camposCondicionalesPersona`) a las personas humanas |
| `inhibicion.admite` | ¿Lleva certificado de inhibición? | Si `true`, mostrá el sub-campo de inhibición |
| `inhibicion.obligatoria` | ¿Es obligatorio? | Si `true`, exigilo |
| `asentimiento` | ¿Las personas de este rol pueden prestar asentimiento? | Marca de apoyo; la regla completa está en `campos.asentimiento` (ver §5.1) |

**En `campos`:** cada campo tiene `aplica` (¿se muestra en este acto?) y `obligatorio`.
- Si `aplica: false` → **no muestres el campo** en el formulario de este acto.
- Si `aplica: true, obligatorio: true` → mostralo y exigilo.
- Si `aplica: true, obligatorio: false` → mostralo opcional (ej. `reconHipoteca`, `afectaciones`).
- `asentimiento` es especial: ver §5.1.

> **Por qué `familia` está en la entrada:** es informativo (agrupa el acto). No lo necesitás para
> armar el formulario — todo lo que precisás está en `partes` y `campos`. Ignoralo si no te sirve.

---

## 5. Las tres reglas que requieren lógica (no solo leer campos)

Estas son las que más se prestan a error. Prestales atención.

### 5.1 Asentimiento conyugal — condicional

El asentimiento lo presta el cónyuge/conviviente de **quien dispone del inmueble**. **Cuál rol
dispone cambia según el acto** — por eso NO lo hardcodees:

- En **compraventa (1028)**: `campos.asentimiento.rol = "wmc"` → lo presta el **transmitente** (el que vende).
- En **hipoteca (1075)**: `campos.asentimiento.rol = "wmt"` → lo presta el **deudor** (el que hipoteca).
- En **cancelación (1020)**: `campos.asentimiento.aplica = false` → **no se pide**.

Lógica del editor:
1. Mirá `campos.asentimiento`. Si `aplica: false` → no pidas asentimiento, terminá.
2. Si `aplica: true`, el rol que dispone es `campos.asentimiento.rol`.
3. `condicional: "estadoCivil"` significa: **pedí el asentimiento solo si la persona de ese rol está
   casada o en unión convivencial** (`estadoCivil` ∈ {CASADO, CASADA, CONVIVIENTE}). Si es soltera/
   divorciada/viuda con bien propio, no corresponde.
4. Cuando corresponde, mostrá los campos de `camposCondicionalesPersona.asentimientoConyugal`
   (`Conyuge`, `Nupcias`) para capturar los datos del cónyuge que asiente.

### 5.2 Proporciones — suman 1 o no

Cada parte con `proporcion.admite: true` muestra un input de proporción por persona.
- Si `proporcion.sumaUno: true` (ej. adquirentes en compraventa) → **validá que las proporciones de
  las personas de ese rol sumen exactamente 1** antes de permitir enviar. Ej.: dos adquirentes 1/2 + 1/2.
- Si `proporcion.sumaUno: false` (ej. donación de parte indivisa, o el transmitente) → **no valides
  la suma**. La proporción puede ser una fracción parcial (se dona/transmite solo una parte).

> La proporción puede venir como texto ("1/2", "4/51 avas partes", "1/4 NP y 1/4 PL"). Si
> `sumaUno: false`, **no la parsees para validar** — se acepta como texto. Si `sumaUno: true`, sí
> necesitás parsear fracciones para verificar la suma.

### 5.3 Roles genéricos — no traducir

Ya dicho pero se repite por lo importante: rotulá las partes con `roles[rol].label`
("Adquirente / Titular", "Transmitente", "Acreedor") en **todos** los actos. No cambies a
nomenclatura por-acto. Es lo que el escribano ya conoce.

---

## 6. Pseudocódigo de referencia — armar el formulario

```
función armarFormulario(codigoActo, artefacto):
    esquema = artefacto.esquema
    acto    = artefacto.actos[codigoActo]
    si acto es null:
        // acto en catálogo pero no habilitado
        mostrar "No disponible por testimonio digital aún"; return

    form = nuevoFormulario(titulo = acto.nombre)

    // --- PARTES ---
    para cada parte en acto.partes:
        grupo = form.agregarGrupoPartes(
            etiqueta   = esquema.roles[parte.rol].label,   // NO traducir
            obligatorio= parte.obligatorio,
            multiple   = (parte.cardinalidad == "1..n")
        )
        // plantilla de persona dentro del grupo:
        grupo.plantillaPersona = funcion(tipoPersona):   // tipoPersona ∈ H/J/O
            campos = copiar(esquema.tiposDePersona[tipoPersona].campos)
            si parte.datosHumanosCompletos y tipoPersona == "H":
                campos += esquema.camposCondicionalesPersona.datosHumanosCompletos
            si parte.proporcion.admite:
                campos += "Proporcion"
            si parte.inhibicion.admite:
                campos += campoCertificado("Inhibicion", obligatorio = parte.inhibicion.obligatoria)
            return renderCampos(campos, labels = esquema.labelsCampo, enums = esquema.enums)

    // --- CAMPOS DE ACTO ---
    para (nombreCampo, def) en acto.campos:
        si nombreCampo == "asentimiento": continue    // se maneja aparte, abajo
        si def.aplica:
            form.agregarCampo(
                etiqueta   = esquema.labelsCampo[nombreCampo],
                tipo       = tipoDeCampoDe(nombreCampo),   // monto / certificado / texto
                obligatorio= def.obligatorio
            )

    // --- ASENTIMIENTO (condicional) ---
    asen = acto.campos.asentimiento
    si asen.aplica:
        rolDisponente = asen.rol
        // en runtime, cuando el escribano cargue las personas de ese rol:
        onCambioEstadoCivil(rolDisponente, persona):
            si persona.EstadoCivil en {CASADO, CASADA, CONVIVIENTE}:
                mostrarCamposAsentimiento(persona, esquema.camposCondicionalesPersona.asentimientoConyugal)
            sino:
                ocultarCamposAsentimiento(persona)

    return form


función validarAntesDeEnviar(form, acto):
    // obligatorios
    para cada parte obligatoria: exigir ≥1 persona
    para cada campo obligatorio con aplica: exigir valor
    // proporciones
    para cada parte con proporcion.admite y proporcion.sumaUno:
        exigir suma(proporciones) == 1
    // asentimiento
    si acto.campos.asentimiento.aplica:
        para cada persona del rol disponente que esté casada/en unión:
            exigir datos de asentimiento
    // ...luego serializar a XML y validar contra el XSD (ver §7)
```

---

## 7. Después del formulario: XML, XSD y envío

El artefacto te ayuda a **armar y pre-validar** el formulario. Pero el testimonio se envía como
**XML v3**, y hay dos pasos más que el artefacto NO cubre:

1. **Serializar a XML v3 y validar contra el XSD.** El XSD es la especificación **precisa** de la
   estructura (tipos exactos, patrones, anidamiento, cardinalidad). El artefacto es una vista
   simplificada para UI; **no reemplaza la validación XSD**. Validá el XML contra el XSD **antes de
   enviar** — el backend lo va a re-validar igual, así que atrapar errores acá te ahorra rechazos.

2. **Firmar (PAdES) y enviar.** El protocolo de envío (endpoint, formato, autenticación, respuesta,
   códigos de error) es un tema aparte de este documento. → *(referenciar el documento de integración
   del protocolo cuando esté; hoy pendiente de documentar)*.

> **Regla mental:** artefacto = **armar** el formulario · XSD = **validar** el XML · protocolo =
> **enviar**. Los tres se usan, en ese orden.

---

## 8. Checklist de integración

- [ ] Descargar y cachear los tres archivos (catálogo, artefacto, XSD).
- [ ] Selector de actos desde el catálogo (231); grisar los que no están en el artefacto.
- [ ] Armar formulario desde el nivel 2 del acto, usando el nivel 1 para las estructuras.
- [ ] Implementar asentimiento condicional (§5.1) — el rol sale del artefacto, no hardcodear.
- [ ] Implementar validación de proporciones según `sumaUno` (§5.2).
- [ ] Rotular partes con los roles genéricos, sin traducir (§5.3).
- [ ] Validar el XML contra el XSD antes de enviar (§7).
- [ ] Detectar cambios de versión: `versionContenido` (refrescar datos) vs `versionEsquema` (revisar).
- [ ] Manejar los códigos de error de rechazo del backend *(cuando esté el doc de protocolo)*.

---

## 9. Preguntas abiertas a coordinar con el RPI

- Formato/endpoint de publicación del artefacto (¿URL fija? ¿cómo se entera el editor de una versión
  nueva — polling, endpoint de versión?).
- Documento del protocolo de envío (endpoint, auth, formato, catálogo de errores) — pendiente.
- Semántica del "aceptado" sincrónico: ¿confirma recepción/validación, o inscripción registral?
