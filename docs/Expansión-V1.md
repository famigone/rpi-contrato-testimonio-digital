# Diseño de la expansión del contrato — alineamiento al Excel del RPI

> **Propósito**: documentar las decisiones de modelado para expandir el contrato
> del Testimonio Digital de manera que cubra todos los campos del Excel
> "EstructuraMinuta" del RPI.
>
> **Estado**: borrador para revisión del responsable del proyecto antes de
> avanzar con la implementación en XSDs y código.
>
> **Audiencia**: equipo técnico del RPI, programador del Colegio (informativo).

---

## Decisiones globales

| # | Tema | Decisión |
|---|------|----------|
| 1 | Persona jurídica | Soportada. CUIT obligatorio, `Documento.numero` repite el CUIT por compatibilidad con el esquema legacy. |
| 2 | Representante | Modelado pero **siempre opcional** en v1.0. Validaciones cruzadas (ej. menor de edad debe tener representante) se hacen en el servicio, no en el XSD. |
| 3 | Asentimiento conyugal | Texto libre estructurado como campo aparte (no como parte de Observaciones). |
| 4 | Reconocimiento de hipoteca y afectaciones al dominio | Texto libre, campos opcionales. |
| 5 | Datos de contacto del rogante | Obligatorios (localidad, provincia, domicilio, teléfono). |
| 6 | Visado de Rentas | Obligatorio. Si tiene visado (R), el número de trámite también es obligatorio. |
| 7 | Tipos de documento | Alineados al Excel: DNI, LE, LC, PAS. **Sin CI**. |
| 8 | Tipos de persona | H (humana), J (jurídica), O (organismo público). |
| 9 | Repositorio del servicio | `rpi-td` (no `rpi-tdigital`). |

---

## 1. Estructura general del XML expandido

```xml
<TestimonioDigital>

  <MetadatosEnvio>
    <!-- igual que v0.draft: identificador, timestamp, hash PDF, versión -->
  </MetadatosEnvio>

  <EscribanoAutorizante>
    <Nombre>...</Nombre>
    <CUIT>...</CUIT>
    <RegistroNumero>...</RegistroNumero>
    <Sede>...</Sede>
  </EscribanoAutorizante>

  <Otorgamiento>
    <Lugar>...</Lugar>
    <Fecha>...</Fecha>
    <NumeroEscritura>...</NumeroEscritura>
    <FolioProtocolo>...</FolioProtocolo>
  </Otorgamiento>

  <CertificacionRegistralPrevia>
    <Numero>...</Numero>
    <FechaEmisionPrimera>...</FechaEmisionPrimera>      <!-- antes "FechaEmision" -->
    <FechaEmisionSegunda>...</FechaEmisionSegunda>      <!-- antes "FechaVigencia" -->
  </CertificacionRegistralPrevia>

  <CertificacionCatastral>                              <!-- 🆕 nuevo bloque -->
    <Emitido>true|false</Emitido>
    <Numero>...</Numero>                                <!-- si Emitido=true -->
    <CodigoValidacion>...</CodigoValidacion>            <!-- si Emitido=true -->
    <TieneObservaciones>true|false</TieneObservaciones>
    <Observaciones>...</Observaciones>                  <!-- si TieneObservaciones=true -->
  </CertificacionCatastral>

  <NomenclaturaCatastral>                               <!-- 🆕 nuevo bloque -->
    <Campo1>..</Campo1>                                 <!-- 2 chars -->
    <Campo2>..</Campo2>                                 <!-- 2 chars -->
    <Campo3>...</Campo3>                                <!-- 3 chars -->
    <Campo4>....</Campo4>                               <!-- 4 chars -->
    <Campo5>....</Campo5>                               <!-- 4 chars -->
  </NomenclaturaCatastral>

  <DatosEconomicos>
    <ValuacionFiscal>
      <valor>...</valor>
    </ValuacionFiscal>
    <Monto>
      <valor>...</valor>
      <moneda>$|USD</moneda>
      <cotizacion>...</cotizacion>                      <!-- si moneda=USD -->
    </Monto>
  </DatosEconomicos>

  <VisadoRentas>                                        <!-- 🆕 nuevo bloque -->
    <Tipo>R|A</Tipo>
    <NumeroTramite>...</NumeroTramite>                  <!-- si Tipo=R -->
  </VisadoRentas>

  <Acto>
    <Compraventa>
      <DescripcionActoIncompleto>...</DescripcionActoIncompleto> <!-- 🆕 opcional -->
      <Adquirentes>
        <Persona> ... </Persona>                        <!-- 1 a N -->
      </Adquirentes>
      <Transmitentes>
        <Persona> ... </Persona>                        <!-- 1 a N -->
      </Transmitentes>
      <Inmuebles>
        <Inmueble> ... </Inmueble>                      <!-- 1 a N -->
      </Inmuebles>
      <ReconocimientoHipotecaMedidasCautelares>...</ReconocimientoHipotecaMedidasCautelares>  <!-- 🆕 opcional, texto libre -->
      <AfectacionesAlDominio>...</AfectacionesAlDominio> <!-- 🆕 opcional, texto libre -->
      <AsentimientoConyugal>...</AsentimientoConyugal>  <!-- 🆕 opcional, texto libre -->
    </Compraventa>
  </Acto>

  <Rogante>
    <CUIT>...</CUIT>
    <Nombre>...</Nombre>
    <NumeroRegistro>...</NumeroRegistro>                <!-- 🆕 obligatorio -->
    <Localidad>...</Localidad>                          <!-- 🆕 obligatorio -->
    <Provincia>...</Provincia>                          <!-- 🆕 obligatorio -->
    <Domicilio>...</Domicilio>                          <!-- 🆕 obligatorio -->
    <Telefono>...</Telefono>                            <!-- 🆕 obligatorio -->
  </Rogante>

  <Observaciones>...</Observaciones>                    <!-- opcional, texto libre -->

  <ds:Signature> ... </ds:Signature>
</TestimonioDigital>
```

---

## 2. Detalle de cada bloque

### 2.1 MetadatosEnvio (sin cambios)

Igual a la versión draft anterior.

### 2.2 EscribanoAutorizante (ajustes de longitud)

| Campo | Tipo XSD | Longitud max | Obligatorio | Correspondencia legacy |
|-------|----------|--------------|-------------|------------------------|
| Nombre | string | **60** (antes 120) | sí | `wmm.wmmeau` |
| CUIT | CUITType | 13 con guiones | sí | `wmm.wmmeac` |
| RegistroNumero | string | **8** (antes 10) | sí | `wmm.wmmear` |
| Sede | string | **40** (antes 60) | sí | `wmm.wmmeas` |

### 2.3 Otorgamiento (sin cambios)

Igual a la versión draft anterior.

### 2.4 CertificacionRegistralPrevia (renombre de subcampos)

| Campo | Tipo XSD | Obligatorio | Correspondencia legacy | Notas |
|-------|----------|-------------|------------------------|-------|
| Numero | string (max **40**) | sí | `wmm.wmmcrn` | (antes max 30) |
| FechaEmisionPrimera | xs:date | sí | `wmm.wmmcrf` | (antes "FechaEmision") |
| FechaEmisionSegunda | xs:date | sí | `wmm.wmmcf2` | (antes "FechaVigencia"). Nombre alineado al Excel. |

### 2.5 🆕 CertificacionCatastral (nuevo)

| Campo | Tipo XSD | Longitud | Obligatorio | Correspondencia legacy |
|-------|----------|----------|-------------|------------------------|
| Emitido | xs:boolean | - | sí | `wmm.wmmadc` ('S'/'N') |
| Numero | string | 40 | obligatorio si Emitido=true | `wmm.wmmcca` |
| CodigoValidacion | string | 40 | obligatorio si Emitido=true | `wmm.wmmcva` |
| TieneObservaciones | xs:boolean | - | obligatorio si Emitido=true | `wmm.wmmtoc` ('S'/'N') |
| Observaciones | string | 2000 | obligatorio si TieneObservaciones=true | `wmm.wmmobc` |

**Decisión**: el XSD no valida las dependencias condicionales (XSD 1.0 no las soporta limpiamente). El servicio del RPI valida en código.

### 2.6 🆕 NomenclaturaCatastral (nuevo)

Cinco campos fijos de longitud específica:

| Campo | Tipo XSD | Longitud | Obligatorio | Correspondencia legacy |
|-------|----------|----------|-------------|------------------------|
| Campo1 | string | 2 | sí | `wmm.wmmno1` |
| Campo2 | string | 2 | sí | `wmm.wmmno2` |
| Campo3 | string | 3 | sí | `wmm.wmmno3` |
| Campo4 | string | 4 | sí | `wmm.wmmno4` |
| Campo5 | string | 4 | sí | `wmm.wmmno5` |

**Decisión a confirmar**: ¿el bloque entero es opcional, o son todos los campos obligatorios? El Excel no lo aclara. Mi sugerencia: **bloque entero opcional** (no todas las compraventas tienen nomenclatura catastral disponible al momento del acto).

### 2.7 DatosEconomicos (sin cambios estructurales)

Confirmado el formato exacto de moneda según Excel: `$` (peso) y `USD` (dólar).

### 2.8 🆕 VisadoRentas (nuevo)

| Campo | Tipo XSD | Valores | Obligatorio | Correspondencia legacy |
|-------|----------|---------|-------------|------------------------|
| Tipo | enum | 'R' (con visado) / 'A' (sin visado) | sí | `wmm.wmmvis` |
| NumeroTramite | string max 10 | - | obligatorio si Tipo=R | `wmm.wmmvin` |

### 2.9 Acto / Compraventa (campos nuevos dentro)

Nuevos subcampos opcionales:

| Campo | Tipo XSD | Longitud | Correspondencia legacy |
|-------|----------|----------|------------------------|
| DescripcionActoIncompleto | string | 106 | `wmm.wmmesp` |
| ReconocimientoHipotecaMedidasCautelares | string | 4000 | `wmm.wmmmrh` |
| AfectacionesAlDominio | string | 4000 | `wmm.wmmado` |
| AsentimientoConyugal | string | 4000 | `wmm.wmmase` |

Para los textos libres tipo `text` del Excel, fijo límite de **4000 caracteres** en el XSD. El legacy soporta más pero conviene poner un tope para evitar payloads excesivos.

### 2.10 Persona (expansión grande)

Tipo reutilizable para adquirentes y transmitentes. Cambia significativamente.

#### 2.10.1 Discriminación por tipo

Tipo de persona:

| Valor | Descripción | Correspondencia legacy |
|-------|-------------|------------------------|
| H | Persona humana | `wmt.wmttip='H'` / `wmc.wmctip='H'` |
| J | Persona jurídica | `wmt.wmttip='J'` / `wmc.wmctip='J'` |
| O | Organismo público | `wmt.wmttip='O'` / `wmc.wmctip='O'` |

#### 2.10.2 Campos comunes a todos los tipos

| Campo | Tipo XSD | Longitud | Obligatorio | Correspondencia legacy (adq/transm) |
|-------|----------|----------|-------------|--------------------------------------|
| Tipo | enum H/J/O | 1 | sí | `wmttip` / `wmctip` |
| ApellidoODenominacion | string | 180 | sí | `wmtape` / `wmcayn` |
| CUIT | CUITType | 13 | sí | `wmtccc` / `wmcccc` |
| TipoDocumento | enum DNI/LE/LC/PAS | 3 | sí | `wmttdo` / `wmctdo` |
| NumeroDocumento | string | 20 | sí | `wmtdoc` / `wmcdoc` |
| PEP | xs:boolean | - | sí | `wmtpep` / `wmcpep` ('S'/'N') |

**Importante (corrección al diseño previo)**: para transmitentes, los campos `wmcayn` (apellido) y `wmcnom` (nombre) viven en columnas **separadas** en `sisrpi.wmc`. El XML los modela como `ApellidoODenominacion` y `Nombres` separados. No se concatenan.

Para persona jurídica (`Tipo=J`):
- `ApellidoODenominacion` se usa para la **razón social**.
- `NumeroDocumento` repite el CUIT (decisión del usuario).

#### 2.10.3 Campos solo para persona humana (Tipo=H)

Estos campos **deben estar presentes si Tipo=H**, deben omitirse si Tipo=J u O:

| Campo | Tipo XSD | Longitud | Obligatorio (si H) | Correspondencia legacy |
|-------|----------|----------|---------------------|------------------------|
| Nombres | string | 80 | sí | `wmtnom` / `wmcnom` |
| EstadoCivil | enum | 20 | sí (solo adquirente) | `wmteci` |
| Nupcias | string | 10 | opcional | `wmtnup` |
| Conyuge | string | 80 | opcional | `wmtcon` |
| Nacionalidad | string | 20 | sí (solo adquirente) | `wmtnac` |
| FechaNacimiento | xs:date | - | sí (solo adquirente) | `wmtfna` |

**Decisión**: para **transmitente humano**, los campos `EstadoCivil`, `Nacionalidad`, `FechaNacimiento` son **opcionales** (el Excel no aclara, pero conceptualmente para transmitentes la info es menos crítica). Para **adquirente humano**, son **obligatorios**.

#### 2.10.4 Campos solo para persona jurídica u organismo (Tipo=J u O)

| Campo | Tipo XSD | Longitud | Obligatorio (si J/O) | Correspondencia legacy |
|-------|----------|----------|----------------------|------------------------|
| InscripcionOrganismoSede | string | 80 | sí | `wmtins` (solo en adquirentes) |

**Decisión**: para transmitentes J/O, no hay campo equivalente a `wmtins` en `sisrpi.wmc`. Si el transmitente es J/O y tiene inscripción, **se documenta en `Observaciones` o no se modela**.

Mejor opción: **modelar `InscripcionOrganismoSede` también para transmitentes en el XML**, y si el campo no tiene correspondencia en `wmc`, el worker lo descarta o lo guarda en un campo de notas. Esta decisión se cierra en el mapeo a legacy.

#### 2.10.5 Solo para adquirentes

| Campo | Tipo XSD | Formato | Obligatorio | Correspondencia legacy |
|-------|----------|---------|-------------|------------------------|
| Proporcion | string | "N/D" fracción | sí | `wmtpor` |

Formato: string tipo `"1/1"`, `"1/2"`, `"1/5"`, etc. Validación con pattern XSD: `\d+/\d+`. La validación matemática (que las proporciones sumen 1) se hace en el servicio.

**Decisión**: cambio de la versión anterior. Antes modelé `Proporcion` con atributos `numerador` y `denominador`. **Ahora lo dejo como string fracción según Excel** (`wmtpor varchar(20)`). Es más simple y respeta el legacy.

#### 2.10.6 Representante (opcional, bloque anidado)

Si la persona tiene representante, se incluye este sub-bloque:

```xml
<Representante>
  <Apellido>...</Apellido>
  <Nombres>...</Nombres>
  <CUIT>...</CUIT>
  <TipoDocumento>DNI|LE|LC</TipoDocumento>  <!-- ojo: NO acepta PAS según Excel para representante -->
  <NumeroDocumento>...</NumeroDocumento>
  <PEP>true|false</PEP>
</Representante>
```

| Campo | Tipo XSD | Longitud | Obligatorio (si hay representante) | Correspondencia legacy (adq) | Correspondencia legacy (transm) |
|-------|----------|----------|-------------------------------------|------------------------------|----------------------------------|
| Apellido | string | 80 | sí | `wmtran` | `wmcran` |
| Nombres | string | 80 | sí | `wmtrno` | `wmcrno` |
| CUIT | CUITType | 13 | sí | `wmtrcc` | `wmcrcc` |
| TipoDocumento | enum DNI/LE/LC | 3 | sí | `wmtrtd` | `wmcrtd` |
| NumeroDocumento | string | 20 | sí | `wmtrdo` | `wmcrdo` |
| PEP | xs:boolean | - | sí | `wmtrpe` | `wmcrpe` |

**Nota del Excel**: para representante, los tipos de documento del Excel son solo DNI, LE, LC (sin PAS). Mantengo esa restricción.

#### 2.10.7 Ejemplo de Persona humana adquirente

```xml
<Persona>
  <Tipo>H</Tipo>
  <ApellidoODenominacion>GONZÁLEZ</ApellidoODenominacion>
  <Nombres>MARÍA LAURA</Nombres>
  <CUIT>27-28456789-3</CUIT>
  <TipoDocumento>DNI</TipoDocumento>
  <NumeroDocumento>28456789</NumeroDocumento>
  <EstadoCivil>SOLTERA</EstadoCivil>
  <Nacionalidad>ARGENTINA</Nacionalidad>
  <FechaNacimiento>1981-03-15</FechaNacimiento>
  <PEP>false</PEP>
  <Proporcion>1/1</Proporcion>
</Persona>
```

#### 2.10.8 Ejemplo de Persona jurídica adquirente

```xml
<Persona>
  <Tipo>J</Tipo>
  <ApellidoODenominacion>INMOBILIARIA DEL SUR S.A.</ApellidoODenominacion>
  <CUIT>30-12345678-9</CUIT>
  <TipoDocumento>DNI</TipoDocumento>  <!-- valor placeholder, no aplica realmente -->
  <NumeroDocumento>30123456789</NumeroDocumento>  <!-- repite CUIT sin guiones -->
  <PEP>false</PEP>
  <InscripcionOrganismoSede>IGJ - Inspección General de Justicia, CABA</InscripcionOrganismoSede>
  <Proporcion>1/1</Proporcion>
</Persona>
```

**Pregunta abierta** sobre el caso PJ: el campo `TipoDocumento` no tiene sentido conceptual para PJ. Tres opciones:

- **a)** Hacerlo opcional para PJ. El XSD permite omitirlo si `Tipo=J|O`.
- **b)** Fijarlo a un valor convencional como "DNI" aunque no aplique (pragmático).
- **c)** Agregar un valor "CUIT" o "NA" al enum (cambia el legacy).

Mi sugerencia: **opción (a)**, hacerlo opcional cuando `Tipo=J|O`. Las validaciones cruzadas en el código del servicio.

#### 2.10.9 Estado civil — enum

Mantengo los valores que ya tenía: SOLTERO/SOLTERA/CASADO/CASADA/DIVORCIADO/DIVORCIADA/VIUDO/VIUDA/CONVIVIENTE/OTRO.

El Excel solo dice `varchar(20)`, sin enumerar valores. Pero conviene cerrar el conjunto para evitar variantes ortográficas ("casado" vs "Casado", etc.).

### 2.11 Inmueble (cambio estructural)

#### 2.11.1 Cambios respecto al diseño anterior

| Antes | Ahora | Motivo |
|-------|-------|--------|
| Matricula como string "DD-NNNN" | Matricula como `xs:integer` 1-8 dígitos + Departamento por código numérico | Alineamiento al Excel y al legacy |
| Departamento como string libre | Departamento como código 1-16 (enum según tabla) | Alineamiento al Excel |
| Sin Tomo/Folio/Finca | Agregar (opcionales, para inmuebles antiguos sin matrícula) | Alineamiento al Excel |
| Barrio | Barra (string 5 chars) | El Excel dice `wmfbar` = "Barra". El nombre anterior estaba mal. |

#### 2.11.2 Estructura

```xml
<Inmueble>
  <Matricula>3456</Matricula>                    <!-- numérico, opcional -->
  <Barra>A</Barra>                                <!-- 5 chars max, opcional -->
  <Departamento>5</Departamento>                  <!-- código 1-16, obligatorio -->
  <Tomo>...</Tomo>                                <!-- 10 chars, opcional -->
  <Folio>...</Folio>                              <!-- 10 chars, opcional -->
  <Finca>...</Finca>                              <!-- 10 chars, opcional -->
</Inmueble>
```

**Regla de negocio**: el inmueble debe tener **o** matrícula **o** la terna tomo/folio/finca. El XSD no valida esto (es regla cruzada), lo valida el servicio.

#### 2.11.3 Tabla de departamentos (enum)

| Código | Nombre |
|--------|--------|
| 1 | Aluminé |
| 2 | Añelo |
| 3 | Catán Lil |
| 4 | Collón Curá |
| 5 | Confluencia |
| 6 | Chos Malal |
| 7 | Huiliches |
| 8 | Lácar |
| 9 | Loncopué |
| 10 | Los Lagos |
| 11 | Minas |
| 12 | Ñorquín |
| 13 | Pehuenches |
| 14 | Picunches |
| 15 | Picún Leufú |
| 16 | Zapala |

Modelado como `xs:integer` con `enumeration` de 1 a 16 en el XSD.

### 2.12 Rogante (expansión)

| Campo | Tipo XSD | Longitud | Obligatorio | Correspondencia legacy |
|-------|----------|----------|-------------|------------------------|
| CUIT | CUITType | 13 | sí | (asumido del escribano si coincide) |
| Nombre | string | 120 | sí | (asumido del escribano si coincide) |
| NumeroRegistro | string | 20 | sí | `wmm.wmmreg` |
| Localidad | string | 30 | sí | `wmm.wmmloc` |
| Provincia | string | 20 | sí | `wmm.wmmprv` |
| Domicilio | string | 40 | sí | `wmm.wmmdom` |
| Telefono | string | 20 | sí | `wmm.wmmtel` |

### 2.13 Observaciones (sin cambios)

Texto libre opcional. Distinto de `AsentimientoConyugal`, `ReconocimientoHipoteca...` y `AfectacionesAlDominio` — esos están dentro de `<Compraventa>` con su propio campo.

Correspondencia legacy: `wmm.wmmobs`.

---

## 3. Resumen de campos del Excel mapeados al XML

### 3.1 Tabla A — Datos generales

| Campo legacy | XML correspondiente |
|--------------|---------------------|
| `wmmwmm` | (interno, no en XML) |
| `wmmac1` (1028 fijo) | (implícito por estar en `<Compraventa>`) |
| `wmmtip` ('N' fijo) | (implícito, todos los testimonios son notariales en v1) |
| `wmmesp` | `Acto/Compraventa/DescripcionActoIncompleto` |
| `wmmno1..no5` | `NomenclaturaCatastral/Campo1..Campo5` |
| `wmmoto` | `Otorgamiento/Lugar` |
| `wmmotf` | `Otorgamiento/Fecha` |
| `wmmnuc` | `Otorgamiento/NumeroEscritura` |
| `wmmfoc` | `Otorgamiento/FolioProtocolo` |
| `wmmeau` | `EscribanoAutorizante/Nombre` |
| `wmmear` | `EscribanoAutorizante/RegistroNumero` |
| `wmmeas` | `EscribanoAutorizante/Sede` |
| `wmmeac` | `EscribanoAutorizante/CUIT` |
| `wmmmo1` | `DatosEconomicos/Monto/valor` |
| `wmmpd1` | `DatosEconomicos/Monto/moneda` |
| `wmmco1` | `DatosEconomicos/Monto/cotizacion` |
| `wmmmo2` | `DatosEconomicos/ValuacionFiscal/valor` |
| `wmmcrn` | `CertificacionRegistralPrevia/Numero` |
| `wmmcrf` | `CertificacionRegistralPrevia/FechaEmisionPrimera` |
| `wmmcf2` | `CertificacionRegistralPrevia/FechaEmisionSegunda` |
| `wmmadc` | `CertificacionCatastral/Emitido` |
| `wmmcca` | `CertificacionCatastral/Numero` |
| `wmmcva` | `CertificacionCatastral/CodigoValidacion` |
| `wmmtoc` | `CertificacionCatastral/TieneObservaciones` |
| `wmmobc` | `CertificacionCatastral/Observaciones` |
| `wmmmrh` | `Acto/Compraventa/ReconocimientoHipotecaMedidasCautelares` |
| `wmmado` | `Acto/Compraventa/AfectacionesAlDominio` |
| `wmmase` | `Acto/Compraventa/AsentimientoConyugal` |
| `wmmobs` | `Observaciones` |
| `wmmvis` | `VisadoRentas/Tipo` |
| `wmmvin` | `VisadoRentas/NumeroTramite` |
| `wmmrog` | (resuelto por el servicio a partir de `Rogante/CUIT`) |
| `wmmreg` | `Rogante/NumeroRegistro` |
| `wmmloc` | `Rogante/Localidad` |
| `wmmprv` | `Rogante/Provincia` |
| `wmmdom` | `Rogante/Domicilio` |
| `wmmtel` | `Rogante/Telefono` |
| `wmmpwd` | (interno: lo setea el servicio) |

### 3.2 Tabla B — Adquirentes

| Campo legacy | XML correspondiente |
|--------------|---------------------|
| `wmtwmt` | (interno) |
| `wmtwmm` | (interno) |
| `wmtite` | (implícito por orden en `Adquirentes`) |
| `wmttip` | `Persona/Tipo` |
| `wmtape` | `Persona/ApellidoODenominacion` |
| `wmtnom` | `Persona/Nombres` (solo si Tipo=H) |
| `wmtccc` | `Persona/CUIT` |
| `wmttdo` | `Persona/TipoDocumento` |
| `wmtdoc` | `Persona/NumeroDocumento` |
| `wmteci` | `Persona/EstadoCivil` |
| `wmtnup` | `Persona/Nupcias` |
| `wmtcon` | `Persona/Conyuge` |
| `wmtnac` | `Persona/Nacionalidad` |
| `wmtfna` | `Persona/FechaNacimiento` |
| `wmtpep` | `Persona/PEP` |
| `wmtins` | `Persona/InscripcionOrganismoSede` |
| `wmtpor` | `Persona/Proporcion` |
| `wmtran` | `Persona/Representante/Apellido` |
| `wmtrno` | `Persona/Representante/Nombres` |
| `wmtrcc` | `Persona/Representante/CUIT` |
| `wmtrtd` | `Persona/Representante/TipoDocumento` |
| `wmtrdo` | `Persona/Representante/NumeroDocumento` |
| `wmtrpe` | `Persona/Representante/PEP` |

### 3.3 Tabla C — Transmitentes

| Campo legacy | XML correspondiente |
|--------------|---------------------|
| `wmcwmc` | (interno) |
| `wmcwmm` | (interno) |
| `wmctip` | `Persona/Tipo` |
| `wmcayn` | `Persona/ApellidoODenominacion` |
| `wmcnom` | `Persona/Nombres` |
| `wmcccc` | `Persona/CUIT` |
| `wmctdo` | `Persona/TipoDocumento` |
| `wmcdoc` | `Persona/NumeroDocumento` |
| `wmcpep` | `Persona/PEP` |
| `wmcran..wmcrpe` | `Persona/Representante/...` |

**Nota**: para transmitentes, el Excel **no incluye** `EstadoCivil`, `Nupcias`, `Conyuge`, `Nacionalidad`, `FechaNacimiento`, `Proporcion`, `InscripcionOrganismoSede`. En el XML, si la persona es un transmitente, esos campos son **opcionales** (no se exigen). Si el sistema del Colegio los completa igualmente, el RPI los recibe pero los descarta al mapear a `wmc`.

### 3.4 Tabla D — Inmueble

| Campo legacy | XML correspondiente |
|--------------|---------------------|
| `wmfwmf` | (interno) |
| `wmfwmm` | (interno) |
| `wmfmat` | `Inmueble/Matricula` |
| `wmfbar` | `Inmueble/Barra` |
| `wmfdep` | `Inmueble/Departamento` |
| `wmftom` | `Inmueble/Tomo` |
| `wmffol` | `Inmueble/Folio` |
| `wmffin` | `Inmueble/Finca` |

---

## 4. Validaciones cruzadas (en código, no en XSD)

Estas reglas las valida el servicio del RPI, no el XSD. Si fallan, el servicio responde con un código de error específico.

| Regla | Código de error sugerido |
|-------|--------------------------|
| Si moneda=USD entonces cotización obligatoria | `MONEDA_SIN_COTIZACION` |
| Si Persona/Tipo=H entonces Nombres obligatorio | `PERSONA_HUMANA_SIN_NOMBRES` |
| Si Persona/Tipo=J u O entonces Nombres se ignora | (warning, no error) |
| Si CertificacionCatastral/Emitido=true entonces Numero y CodigoValidacion obligatorios | `CERT_CATASTRAL_DATOS_FALTANTES` |
| Si CertificacionCatastral/TieneObservaciones=true entonces Observaciones obligatorio | `CERT_CATASTRAL_OBS_FALTANTE` |
| Si VisadoRentas/Tipo=R entonces NumeroTramite obligatorio | `VISADO_SIN_NUMERO_TRAMITE` |
| Inmueble debe tener Matricula O (Tomo Y Folio Y Finca) | `INMUEBLE_SIN_IDENTIFICACION` |
| Suma de proporciones de Adquirentes debe ser 1 | `PROPORCIONES_NO_SUMAN_UNO` |
| El CUIT del firmante XML-DSig debe coincidir con `EscribanoAutorizante/CUIT` | `CUIT_FIRMANTE_NO_COINCIDE` |

---

## 5. Limitaciones que se mantienen en v1.0 (expansión)

A pesar de la expansión, todavía hay cosas que **no se modelan**:

- **Múltiples actos en un mismo testimonio**: un testimonio = un acto (compraventa en v1).
- **Otros tipos de actos**: hipoteca, donación, división de condominio, etc. En v1 solo compraventa.
- **Asentimiento conyugal estructurado**: se modela como **texto libre** (campo `AsentimientoConyugal`). Si en el futuro se necesita estructurar (cónyuge identificado, fecha, etc.), evolución v2.

Estas limitaciones se documentan en `docs/01-introduccion.md`.

---

## 6. Impacto en otros documentos

### En el repo del contrato (`rpi-contrato-testimonio-digital`)

- `docs/01-introduccion.md`: reescribir sección "Limitaciones de v1.0" — ahora son menos.
- `docs/04-formato-xml.md`: reescribir significativamente para mostrar la estructura nueva.
- `docs/07-respuestas-y-errores.md`: agregar los códigos de error de validaciones cruzadas.
- `docs/09-glosario.md`: agregar términos nuevos (Visado de Rentas, Nomenclatura Catastral, Representante, etc.).
- `docs/10-campos-del-formulario.md`: **NUEVO**, tabla plana de todos los campos para el programador del Colegio.
- `CHANGELOG.md`: registrar la expansión.
- `xsd/comunes/persona-humana.xsd` → renombrar a `xsd/comunes/persona.xsd`. Expandir.
- `xsd/comunes/identificacion-inmueble.xsd`: cambiar estructura completa.
- `xsd/comunes/certificacion-registral.xsd`: ajustar nombres de subcampos.
- `xsd/comunes/datos-economicos.xsd`: sin cambios.
- `xsd/comunes/escribano-autorizante.xsd`: ajustar longitudes.
- `xsd/comunes/metadatos-envio.xsd`: sin cambios.
- `xsd/comunes/otorgamiento.xsd`: sin cambios.
- `xsd/comunes/rogante.xsd`: expandir con datos de contacto.
- `xsd/comunes/certificacion-catastral.xsd`: **NUEVO**.
- `xsd/comunes/nomenclatura-catastral.xsd`: **NUEVO**.
- `xsd/comunes/visado-rentas.xsd`: **NUEVO**.
- `xsd/comunes/representante.xsd`: **NUEVO**.
- `xsd/actos/compraventa.xsd`: expandir con campos nuevos.
- `xsd/testimonio-digital.xsd`: ajustar includes.
- `ejemplos/`: actualizar los 3 existentes con la estructura nueva. Agregar 3 más:
  - `compraventa-persona-juridica.xml`.
  - `compraventa-con-representante.xml`.
  - `compraventa-inmueble-antiguo.xml` (sin matrícula, con tomo/folio/finca).

### En el repo del servicio (`rpi-td`)

- `TD-Mapeo-XML-Legacy.md`: reescribir con el mapeo nuevo (corrigiendo `wmcayn` y agregando todos los campos nuevos).
- Parser de Fase 2: extraer los campos nuevos del XML y persistirlos en `td_contenidos.datos_parseados`.
- Worker de Fase 3: mapear los campos nuevos a las columnas correspondientes de `wmm`, `wmt`, `wmc`, `wmf`.
- Validaciones cruzadas: agregar las reglas de §4.
- Tests: agregar casos para los escenarios nuevos.

---

## 7. Preguntas abiertas para confirmar

Antes de implementar, conviene confirmar:

| # | Pregunta | Mi sugerencia |
|---|----------|---------------|
| 1 | `NomenclaturaCatastral`: ¿bloque obligatorio o opcional? | Opcional. |
| 2 | `TipoDocumento` para PJ/Organismo: ¿opcional o placeholder? | Opcional. |
| 3 | `InscripcionOrganismoSede` para transmitentes J/O: ¿modelar y descartar al mapear, o no modelar? | Modelar (más prolijo desde el XML). |
| 4 | Límite de texto libre (asentimiento, observaciones, etc.): ¿4000 caracteres es razonable? | 4000 es buen tope. |
| 5 | Para transmitente humano: ¿`EstadoCivil`, `Nacionalidad`, `FechaNacimiento` opcionales? | Sí, opcionales. |
| 6 | `Proporcion` como string fracción (`"1/2"`) vs atributos `numerador`/`denominador` | String fracción, alineado al Excel. |

---

## 8. Plan de implementación (después de aprobación)

Una vez aprobado este diseño, las fases siguientes son:

**Fase 2** (Claude Code en repo del contrato): implementar XSDs, ejemplos, docs.

**Fase 3** (Claude Code en repo del servicio): actualizar mapeo, parser, worker, tests.

Cada fase tiene sus propias instrucciones detalladas que armo después de tu OK a este diseño.