# Changelog

Todas las modificaciones notables a este contrato se documentan acá.

El formato sigue [Keep a Changelog 1.1.0](https://keepachangelog.com/es-ES/1.1.0/)
y este contrato adhiere a [Semantic Versioning 2.0.0](https://semver.org/spec/v2.0.0.html).

---

## [3.2.2] — 2026-10-26

Solo **documentación**: ningún XSD y ningún comportamiento cambian. El capítulo
07 pasa a describir **lo que el servicio devuelve hoy**, verificado contra el
código. Hasta ahora prometía otra cosa en varios puntos, y uno de ellos rompía
el manejo de errores de cualquier cliente que lo siguiera al pie de la letra.

### Cambiado
- **El cuerpo del error era otro.** El capítulo lo documentaba anidado,
  `{ "error": { "codigo", "mensaje", … } }`, y el servicio devuelve uno
  **plano**: `error` es un **texto** y el dato sobre el que conviene ramificar
  es **`motivo`**. Un cliente que leyera `error.codigo` obtenía `undefined` en
  **todos** los rechazos. Se corrigió la sección de formato, con la tabla de los
  cinco campos, y los tres ejemplos del final.
  > La **única** excepción es el **429**, que sí usa el sobre anidado. Queda
  > documentado, con la recomendación de manejar las dos formas.
- **Los errores de firma y de certificado son 400, no 401.** El 401 sale
  **únicamente** de la clave de API. Afecta a `FIRMA_INVALIDA`,
  `FIRMA_FALTANTE`, `CERTIFICADO_NO_VIGENTE` y `CERTIFICADO_NO_RECONOCIDO`.
- **La tabla de códigos suma una columna, "Qué viaja en la respuesta"**, con el
  `motivo` y la `causa` de cada caso. La columna `Código` sigue siendo el nombre
  con el que este documento se refiere al error; lo que viaja es el `motivo`.
- **`FIRMA_FALTANTE`** describía "no se encontró bloque `Signature` en el XML",
  que es de antes de la v3.1. Ahora es la falta de la parte
  `firma_xml_escribano`; un `<ds:Signature>` dentro del XML se **rechaza**.
- **La sección de 422** pasa a aclarar que **el RPI no devuelve 422**: las
  reglas del contrato que el XSD no expresa viajan en un **400** con
  `motivo: "estructura_invalida"` y la lista completa en `info.reglas_fallidas`.
  Los códigos de esa sección son los valores de `regla`.
- **`CODIGO_ACTO_NO_HABILITADO`** viaja como `motivo: "codigo_no_habilitado"`
  en un 400, no como una regla.
- **`ROGANTE_NO_ENCONTRADO`** no es un rechazo de la recepción: el envío se
  acepta y el fallo aparece después, al sincronizar, si el CUIT del escribano no
  está dado de alta.
- **5xx**: `BD_NO_DISPONIBLE` viaja como `infra_error`; se documenta
  `VALIDACION_NO_DISPONIBLE` (503), que existe y dice explícitamente que **no**
  se reintente.

### Reservado, no implementado
Se marcan así, **sin borrarlos**, los códigos que el servicio no puede devolver
hoy. Cada uno dice qué pasa en su lugar:

- **`CERTIFICADO_REVOCADO`** — el RPI **no consulta revocación**: sin OCSP ni
  CRL, por decisión de diseño del **ADR-003**. Un certificado revocado pero
  vigente y con cadena válida hoy pasa.
- **`IDENTIFICADOR_DUPLICADO_CONTENIDO_DISTINTO`** (409) — un
  `identificadorEnvio` repetido devuelve 200 con `yaProcesadoAnteriormente`, sin
  comparar el contenido.
- **`SIN_PERMISO`** (403) — no hay permisos por cliente.
- **`TIMEOUT_INTERNO`** (504) — una demora termina en 503 o en 500.
- **`ESCRIBANO_NO_REGISTRADO`**, **`MATRICULA_NO_EXISTENTE`**,
  **`CERTIFICACION_VENCIDA`** — no se verifican en la recepción.
- **`VERSION_CONTRATO_NO_SOPORTADA`** — lo rechaza el XSD, así que llega como
  `estructura_invalida`.

---

## [3.2.1] — 2026-10-26

Solo **documentación**: no cambia ningún XSD ni ningún comportamiento. Documenta
el control del CUIT del firmante, que el servicio ya hace.

### Agregado
- **`docs/07`**, sección "El CUIT del firmante — causas de `firma_invalida`":
  el RPI verifica que quien firmó sea el escribano autorizante declarado,
  comparando el CUIT del certificado con `EscribanoAutorizante/CUIT`. El control
  es el mismo y **estricto** en las dos firmas, la del XML y la PAdES del PDF.

  Se documentan las dos **causas** que viajan bajo `motivo: "firma_invalida"`:
  `cuit_firmante_ilegible` (el `serialNumber` no trae un CUIT/CUIL de 11
  dígitos) y `cuit_firmante_no_coincide` (trae otro). Son dos problemas
  distintos: el primero es del certificado o de cómo la AC cargó el campo, el
  segundo es que la firma no es de quien dice el XML.

  Se documentan además **los formatos que se aceptan** en el `serialNumber`
  —incluida la forma habitual de los certificados de AC-ONTI, `CUIL 20…` y
  `CUIT 30…` sin guiones—, los que no, y **el detalle del rechazo**, que informa
  el número leído y el esperado. El `serialNumber` crudo no se informa: puede
  traer otro documento del firmante.

### Cambiado
- **`docs/07`**, fila `CUIT_NO_COINCIDE`: decía **401** y es **400**. En el
  servicio el 401 sale únicamente de la API key; toda firma rechazada es un 400
  con `motivo: "firma_invalida"`. La fila además apunta ahora a la sección
  nueva, que es donde está el detalle.

---

## [3.2.0] — 2026-10-26

Dos cosas en el mismo MINOR.

**Un elemento nuevo y opcional, `NumeroTasa`.** Los envíos que hoy validan
siguen validando sin tocar nada. Y con él, la corrección de fondo: **la tasa
registral nunca rechaza un ingreso**. La ley impide rechazar un trámite por
cuestiones tributarias, así que lo que la documentación prometía —rechazos por
tasa, un estado intermedio y un rechazo diferido por callback— no existía en el
XSD ni en el servicio, y no va a existir: se elimina.

**Un rechazo nuevo, `CUIT_INVALIDO`.** Un envío con un CUIT mal tipeado que
antes entraba, ahora se rechaza. Entra antes de que el Colegio empiece las
pruebas en serio, y no después, justamente para que no cambie el comportamiento
en medio de ellas.

### Agregado
- **`xsd/v3/testimonio-digital.xsd`**: elemento **`NumeroTasa`**, opcional
  (`minOccurs="0"`), de **primer nivel** y ubicado **después de
  `Otorgamiento`**. Tipo `NumeroTasaType`: `xs:positiveInteger` con
  `totalDigits="12"`.
  > Es de nivel testimonio y no de acto, porque una misma tasa puede cubrir
  > varios actos, cada uno con su concepto y su monto. Uno solo por testimonio.
- **`docs/02-flujo-end-to-end.md`**, sección "La tasa registral no es una
  validación".
- **`docs/03-endpoint-api.md`**, sección "La tasa en la respuesta": el 202
  informa `tasa.numero` y `tasa.estado`, que en ese momento solo puede ser
  `sin_codigo` (el XML no trajo número) o `sin_verificar` (trajo número y
  todavía no se consultó al SUR).
- **`docs/07`**: el código de error **`CUIT_INVALIDO`** (400), con su sección
  propia. El RPI valida el **dígito verificador** (módulo 11) de **todos** los
  CUIT del testimonio: `EscribanoAutorizante`, `Rogante`, cada `Parte` y cada
  `Representante`. El detalle trae **todos** los que fallan —no el primero—, con
  el campo, y para las partes el acto, la posición y el rol.

  Esto implementa lo que el XSD ya documentaba: *"El XSD no valida el dígito
  verificador; lo hace el servicio del RPI"*. Hasta ahora no lo hacía.

  **El prefijo sigue sin importar:** un `30` de persona jurídica es tan válido
  como un `20` o un `27`. El dígito verificador no dice nada del tipo de
  persona, y el RPI no restringe el prefijo en ningún lado.

  Un CUIT que no tenga 11 dígitos NO da este error: eso ya lo rechaza el XSD por
  forma.

### Eliminado
- **Códigos de error `TASA_INEXISTENTE`, `TASA_NO_PAGADA` y
  `TASA_YA_UTILIZADA`** (capítulo 07), con su ejemplo de respuesta.
- **Estados `aceptado_tasa_pendiente` y `rechazado`** (capítulo 03).
- **Evento de callback `validacion_fallida`** (capítulo 08), con su ejemplo y su
  regla de deduplicación propia.
  > Era el **único** rechazo que podía llegar después del 202, y existía solo
  > para el caso de una tasa verificada como impaga. Al desaparecer, **todo
  > rechazo estructural pasa a ser síncrono**: viaja siempre en la respuesta
  > HTTP. Eso simplifica la integración del Colegio, que ya no necesita manejar
  > un rechazo asincrónico.

### Cambiado
- **`docs/07`**, `MULTIPART_INVALIDO`: ahora también cubre **dos partes con el
  mismo nombre**, que antes se aceptaban quedándose con la última en silencio.
  Con la firma detached eso importa más, porque la firma cubre un solo juego de
  bytes y cuál se verificaba quedaba a merced del orden de las partes. El
  detalle trae `causa: "campo_duplicado"` y el nombre del campo repetido.
- **`ejemplos/v3/`** (los doce): se corrige el **dígito verificador** de 53 CUIT.
  Los números siguen siendo los mismos ficticios de siempre, con el último
  dígito arreglado, así que un envío copiado de un ejemplo pasa la validación
  nueva. Verificado que los doce validan contra el XSD de esta versión, el que
  ya tiene `NumeroTasa`.

  En las personas jurídicas se alinea además el `NumeroDocumento`, que por
  contrato **es** el CUIT: al corregir el verificador del CUIT habían quedado
  desalineados, con el ejemplo contradiciéndose a sí mismo.
- **`docs/09-glosario.md`**, "Tasa registral": ahora dice que una tasa impaga
  ingresa y sale provisional, y describe el reingreso por pago. "Rechazo
  estructural" deja de mencionar la tasa y aclara que siempre es síncrono.
- **`docs/02`** y **`docs/03`**: la tasa sale de la lista de validaciones de la
  recepción.

### Qué tiene que hacer el Colegio
- **Nada obligatorio para seguir enviando**: `NumeroTasa` es opcional.
- Para informar la tasa, agregar `<NumeroTasa>` después de `</Otorgamiento>`.
- Si el sistema del Colegio maneja los códigos `TASA_*`, el estado
  `aceptado_tasa_pendiente` o el callback `validacion_fallida`, **se puede
  sacar ese código**: no van a llegar nunca.
- **Revisar el dígito verificador de los CUIT** antes de enviar: desde esta
  versión un CUIT que no cierra se rechaza con `CUIT_INVALIDO`.

---

## [3.1.2] — 2026-10-22

Solo **documentación y reorganización de archivos**: no cambia ningún XSD, así
que no hay nada que migrar. Las correcciones de la firma envuelta son
aclaraciones de cosas que el servicio ya hacía así desde la v3.1.

### Agregado
- **`artefacto-campos-por-acto.json`** (raíz del repo): reglas de **partes y campos por acto
  habilitado** para que el editor del escribano arme el formulario dinámicamente. Versionado propio
  e independiente del contrato: `versionEsquema` (estructuras genéricas; cambiarlo es *breaking*
  para el editor) y `versionContenido` (qué actos están habilitados; no requiere cambios de código).
  > Es un **dato**, no parte del esquema XML: no afecta el XSD ni la versión del
  > contrato. La validación definitiva del testimonio sigue siendo el XSD v3 más
  > las reglas del servicio.
- **`docs/11-artefacto-campos-por-acto.md`**: guía de integración del artefacto con el editor del
  Colegio de Escribanos (antes `INTEGRACION-artefacto-editor.md` en la raíz), incorporada como
  capítulo 11 de la documentación.
- **`docs/README.md`**: índice de la documentación (capítulos, recorridos por perfil y tabla de
  "dónde encontrar cada cosa"). Cada capítulo cierra con un enlace de vuelta al índice.
- **`docs/03-endpoint-api.md`**, sección "Partes del multipart": los nombres de
  campo (`xml`, `pdf`, `firma_xml_escribano`) son **exactos y en minúsculas**.
  El RPI los compara carácter por carácter y distingue mayúsculas: una parte con
  `name="XML"` se descarta en silencio y el envío se rechaza con
  `MULTIPART_INVALIDO` (400) por parte faltante. Se aclara además que el
  `filename` de cada parte el RPI no lo lee nunca.
- **`obsoleto/`** (carpeta nueva, con un README de una línea): el esquema **v1**
  (`testimonio-digital.xsd` de la raíz de `xsd/` más sus `comunes/` y `actos/`) y
  los seis ejemplos sin versión de `ejemplos/`. No se usaban, pero eran los que
  encontraba quien quisiera validar antes de enviar: el XSD v1 exige
  `<ds:Signature>` adentro, así que todo documento válido contra él lo rechaza el
  RPI. `xsd/xmldsig-core-schema.xsd` **no** se movió: v2 y v3 lo importan.

### Cambiado
- **Documentación alineada a v3**: los capítulos 01 a 11, el `README.md`, `xsd/README.md`,
  `ejemplos/README.md` y `xsd/v3/README.md` describen **solo la v3** (namespace `/v3`,
  `version="3.0"`, `<VersionContrato>3.0</VersionContrato>`, XSD `xsd/v3/`). Se quitaron las
  comparaciones con v1/v2; el historial de versiones queda en este CHANGELOG.
- **`docs/07`**: se agrega el código de error `CODIGO_ACTO_NO_HABILITADO` (422).
- **`CONTRIBUTING.md`**: los comandos de validación previos al PR apuntaban a
  `xsd/testimonio-digital.xsd` (v1) y a `ejemplos/*.xml` (sin versión). Ahora
  validan los doce ejemplos de `ejemplos/v3/` contra `xsd/v3/testimonio-digital.xsd`.
- **`ejemplos/README.md`**, sección de firma: describía el bloque
  `<ds:Signature>` con valores placeholder y le indicaba al integrador que su
  librería lo inserta en el documento. Ahora describe el flujo **detached**: XML
  sin firma adentro, firma en su propia parte del multipart.

### Corregido
- **`docs/03-endpoint-api.md`**, Parte 1: decía que el contenido era "el XML del
  testimonio digital, firmado con XML-DSig", que contradecía al resto del
  capítulo y al 05. Desde la **v3.1** el XML va **sin firma adentro** y la firma
  detached viaja en la parte 3; un XML con `<ds:Signature>` adentro se rechaza.
- **`docs/04-formato-xml.md`**: el esqueleto de la raíz mostraba un
  `<ds:Signature>` dentro de `<TestimonioDigital>` — justo lo que el RPI
  rechaza, en el capítulo del que se copia la estructura. Se quita del
  esqueleto, del párrafo de namespaces y de la sección "Signature", que ahora
  dice que el elemento sigue existiendo en el XSD solo por compatibilidad.
- **`ejemplos/v3/`** (los doce archivos): se quita el `<ds:Signature>` envuelto
  con valores placeholder y la declaración `xmlns:ds` que quedaba sin uso. Los
  doce validan contra `xsd/v3/testimonio-digital.xsd`.

---

## [3.1.1] — 2026-10-15

Solo **documentación**: no cambia ningún XSD, así que no hay nada que migrar.
Un cliente ya integrado solo tiene que ajustar su URL base; el resto son
aclaraciones de cosas que ya eran así.

### Cambiado
- **`docs/03-endpoint-api.md`**: se define la URL pública del endpoint.
  - Staging / Homologación: `https://test.jusneuquen.gov.ar/rpi-td/api/v1/testimonios`
  - Producción: `https://[HOST-PRODUCCION-POR-DEFINIR]/rpi-td/api/v1/testimonios`
    (la ruta queda definida; falta confirmar el host)
  Se actualizan el bloque de método HTTP y los dos ejemplos (raw HTTP y `curl`).
- **`README.md`**: en "Decisiones pendientes", la URL de staging pasa a
  resuelta; queda abierto solo el host de producción.

### Agregado
- **`docs/04`**, sección "Orden de armado del envío": las tres partes tienen
  dependencias entre sí, así que el orden no es opcional — (1) firmar el PDF,
  (2) armar el XML completo con ese `HashPDF`, (3) firmar el XML. Se explicitan
  las dos consecuencias: después del paso 3 no se puede tocar nada (el envío es
  un bloque), y **un reintento NO se rearma** — se retransmiten los mismos tres
  archivos, sin regenerar el XML ni refrescar el `TimestampEnvio`.
- **`docs/04`**, sección "Qué es y qué no es `TimestampEnvio`": es el momento de
  **armado** del envío, no el de la transmisión, y queda fijado antes de firmar
  el XML (la firma cubre sus bytes, así que no se puede cambiar después). **El
  momento oficial del envío es la RECEPCIÓN en el RPI**: es la fecha que el RPI
  registra y la que vale a efectos registrales.

  Se documenta además que **el RPI no rechaza por este campo**: no se valida
  contra la hora de recepción, así que un envío armado horas antes o reintentado
  al día siguiente se acepta. Un desfase muy grande (más de 7 días al pasado,
  más de 1 hora al futuro) queda como **advertencia** en la traza, no como
  rechazo.
- **`docs/03`**, sección "Sobre el prefijo `/rpi-td`": el prefijo es parte de la
  **URL pública**, **no del servicio**. Lo agrega el enrutamiento del RPI, cuyo
  proxy inverso lo recorta y le entrega al servicio `/api/v1/testimonios`.

  Se documenta explícitamente porque invita a tres errores:
  1. pegarle a `/api/v1/testimonios` sin el prefijo, que no funciona desde
     afuera;
  2. deducir otras rutas sacando el prefijo — cualquier endpoint futuro se
     publica acá con su URL pública completa;
  3. escribir la URL base en el código. El prefijo es un detalle del
     enrutamiento del RPI y **puede cambiar sin que cambie el contrato del
     testimonio**, así que conviene que sea configurable del lado del
     integrador.

---
## [3.1.0] — 2026-10-13

Cambio **MINOR (compatible)**: se habilita la **firma XAdES-BES detached**, en la que la firma del
escribano viaja como archivo aparte y no dentro del documento XML.

**Por qué**: el firmador del Colegio produce firmas detached, con el digest calculado sobre los
**bytes crudos** del archivo XML. El esquema v3.0 exigía `<ds:Signature>` como último hijo del
elemento raíz, así que un XML con firma detached no validaba.

**Por qué es compatible**: un documento v3.0 con la firma envuelta **sigue siendo válido** contra
este esquema (verificado: los 12 ejemplos de `ejemplos/v3/` validan sin cambios). El atributo
`version` pasa de `fixed="3.0"` a una enumeración `"3.0" | "3.1"`: dejarlo fijo en `3.0` habría
impedido declarar la versión nueva, y fijarlo en `3.1` habría invalidado todos los envíos
existentes, convirtiendo este MINOR en un MAJOR.

### Cambiado
- **`xsd/v3/testimonio-digital.xsd`**: `<ds:Signature>` pasa a `minOccurs="0"`. El esquema deja de
  garantizar por sí solo que el testimonio esté firmado; esa obligación pasa a ser **regla del
  servicio**, que exige la firma siempre y rechaza el envío si falta.
- **`xsd/v3/testimonio-digital.xsd`**: el atributo `version` del elemento raíz pasa de
  `fixed="3.0"` a enumeración `"3.0" | "3.1"`. Los envíos con firma detached declaran `"3.1"`.
- **`docs/03-endpoint-api.md`**: el multipart pasa de dos a **tres partes**. La nueva se llama
  **`firma_xml_escribano`** y es **obligatoria**. Se documenta que el XML se verifica byte a byte y
  se agrega el límite de tamaño de la firma (1 MB).
- **`docs/05-firma-digital.md`**: reescrito para firma detached. Documenta qué cubre cada
  referencia, que el digest del documento es sobre los **bytes exactos** (sin canonicalizar, así que
  CRLF vs LF importa), la política de algoritmos (**SHA-256 o superior; SHA-1 se rechaza**), que el
  CUIT/CUIL se lee del atributo `serialNumber` del certificado aceptando ambos prefijos, y que la
  lista de raíces confiables es configuración del servicio y no parte del contrato.
- **`xsd/v3/comunes/metadatos-envio.xsd`**: se corrige un comentario que seguía diciendo
  `"Para esta v2.0: 2.0"` dentro del esquema v3.

### Agregado
- **`docs/05`**: sección de pendientes de definición — **sello de tiempo (XAdES-T)** y **segunda
  firma (de aplicación, del Colegio)**. Ninguna de las dos es exigible todavía. Se deja constancia
  de que `<xades:SigningTime>` es una fecha **declarada por el firmante** y no prueba cuándo se
  firmó.
- **`docs/05`**: nota para implementadores en Node.js — `xml-crypto` **no resuelve referencias
  externas** y además canonicaliza el contenido referenciado, así que para detached hay que resolver
  la referencia a mano.

### ⚠️ No se acepta la firma envuelta
Aunque el **esquema** sigue admitiendo `<ds:Signature>` dentro del documento (por compatibilidad),
el **servicio** rechaza un XML que la traiga adentro. Un solo modo de firma, para que no haya
ambigüedad sobre qué bytes se verifican.

---

## [3.0.0] — 2026-07-24

Cambio **MAJOR (incompatible)**: el **tipo de acto pasa de ser un elemento del esquema a ser un
DATO**. En v2 cada tipo era un elemento del `xs:choice` de `ActoType` (`<Compraventa/>`,
`<Hipoteca/>`, `<Donacion/>`, `<Permuta/>`) con un `xs:complexType` propio; esos tipos estaban
**vacíos** (solo textos libres). v3 los elimina y modela el acto con **`<Codigo>`** (el código del
catálogo legacy `act`). Namespace nuevo `.../testimonio-digital/v3`, `version="3.0"`. Ver
[ADR-004](https://github.com/) (repo del servicio, `docs/adr/004-contrato-v3-codigo-como-dato.md`).

**Por qué ahora:** hay 4 actos y **cero emisores en producción real** (las compraventas
sincronizadas fueron en *staging*). El breaking change es barato hoy; en seis meses, con emisores
integrados, sería caro.

### ⚠️ Cómo migrar (para sistemas notariales que consumen el contrato)

- **El elemento de acto se reemplaza por `<Codigo>`**, primer hijo del `<Acto>`:
  - `<Compraventa/>` → `<Codigo>1028</Codigo>`
  - `<Hipoteca/>` → `<Codigo>1075</Codigo>`
  - `<Donacion/>` → `<Codigo>1056</Codigo>`
  - `<Permuta/>` → `<Codigo>1102</Codigo>`
- **Los textos libres** que vivían dentro del elemento de acto (`DescripcionActoIncompleto`,
  `ReconocimientoHipotecaMedidasCautelares`, `AfectacionesAlDominio`, `AsentimientoConyugal`)
  ahora son **hijos opcionales de `<Acto>`**, al final (después de `VisadoRentas`).
- **`<ActoSecundario>`** (código, desde v2.4.0) **no cambia**: va después de `<Codigo>`.
- **Namespace y versión**: `xmlns=".../v3"`, `version="3.0"`, `<VersionContrato>3.0</VersionContrato>`,
  `schemaLocation` → `xsd/v3/testimonio-digital.xsd`.

### Agregado
- **`xsd/v3/`**: el contrato v3 completo. `ActoType` con `<Codigo>` (`CodigoActoType`, `xs:integer`
  ≤ 9999 — **NO** un enum de los ~200 códigos, ver abajo) + los textos libres opcionales.
- **`catalogo-actos.json`** (raíz del repo): lista **informativa** de códigos existentes (código +
  nombre). Es **dato, no esquema**: NO valida nada. Compensa la ausencia del enum. Qué códigos están
  *habilitados* lo decide el servicio, no este archivo.
- **`ejemplos/v3/`**: los 11 ejemplos de v2 migrados + **`cancelacion-hipoteca-ejemplo-valido.xml`**
  (código 1020) — un caso que **v2 NO podía expresar** (el catastro era obligatorio) y v3 habilita.

### Cambiado (incompatible)
- **`<Codigo>` es `xs:integer`, no un enum.** Razón: el catálogo legacy cambia (5 altas en 7 meses:
  1326/1327 dic-2025, 1330 mar-2026, 1334/1335 jun-2026). Con enum, cada alta del catálogo sería un
  release del contrato aunque el servicio no habilite ese acto. El contrato no debe versionarse por
  cambios del catálogo. La existencia de códigos se documenta en `catalogo-actos.json`.
- **`CertificacionCatastral` y `NomenclaturaCatastral` pasan a `minOccurs=0`** (eran obligatorias por
  inmueble). La familia HIPOTECARIA-LIBERA (cancelación, liberación) no las lleva; su obligatoriedad
  para las familias que sí las requieren se valida en el servicio (regla `exigeCatastral`). Es la
  **única** obligatoriedad estructural que v3 relaja.

### Eliminado
- **`xsd/v3/actos/`** no existe: los 4 XSD de acto de v2 (`compraventa.xsd`, `hipoteca.xsd`,
  `donacion.xsd`, `permuta.xsd`) no tienen equivalente — el acto ya no tiene tipo XSD propio.

### Sin cambios
- Todo lo demás de v2: modelo de partes con rol, N actos, Inmuebles/DatosEconomicos/VisadoRentas
  obligatorios, acto secundario, componentes comunes (`comunes/`), firma XML-DSig.
- **v2 queda congelado en `xsd/v2/`** (coexiste, como v1 en su momento).

---

## [2.4.0] — 2026-07-23

Cambio **MINOR** (aditivo y retrocompatible): se incorpora el **ACTO SECUNDARIO** de una
minuta. Es una **dimensión del modelo** que faltaba, no un acto nuevo. Mismo namespace y
`version="2.0"`; todos los testimonios existentes siguen validando (el elemento es opcional).

El legacy modela "una minuta = acto principal + acto secundario" (columnas `wmmac1`/`wmmgr1` +
`wmmac2`/`wmmgr2`). El caso frecuente es la **compraventa con hipoteca simultánea** (comprar con
crédito): UN solo acto registral con DOS códigos. Hasta 2.3.0 el contrato no podía expresarlo (un
`<Acto>` lleva un solo tipo por el `xs:choice`), así que ese caso —el más frecuente del registro—
no tenía canal correcto.

### Agregado
- `testimonio-digital.xsd`: elemento **opcional** `<ActoSecundario>` dentro de `ActoType`
  (inmediatamente después del choice del acto principal). Su tipo `CodigoActoSecundarioEnum` está
  **restringido a los dos códigos** que el legacy admite como secundario: **1075** (HIPOTECA) y
  **1157** (RECONOCIMIENTO DE HIPOTECA).
- Es la **primera vez** que el contrato usa un **código** en vez de un nombre de elemento. Está
  justificado: el acto secundario **no tiene estructura propia** (partes, inmuebles y montos son
  compartidos por ambos actos del mismo `<Acto>`), es un **marcador** del segundo código. Anticipa
  la dirección del v3 (actos por código de catálogo). Documentado así en el XSD.
- Ejemplo `ejemplos/v2/compraventa-con-hipoteca-ejemplo-valido.xml` (datos **ficticios**):
  compraventa (principal) + `ActoSecundario=1075`, con ADQUIRENTE + TRANSMITENTE + ACREEDOR y los
  **tres montos** en el mismo acto (precio + valuación + `MontoHipoteca`). El vendedor y el
  acreedor son la misma entidad (I.P.V.U.) con el mismo CUIT en dos roles.

### Sin cambios
- El `xs:choice` del acto **principal** no cambia (Compraventa/Hipoteca/Donacion/Permuta).
- `comunes/datos-economicos.xsd` **no se toca**: los tres bloques económicos (`ValuacionFiscal`,
  `Monto`, `MontoHipoteca`) ya coexistían; una compraventa-con-hipoteca los usa a los tres.
- Qué acto principal admite secundario, y las reglas que el secundario dispara (1075 exige
  `MontoHipoteca` y rol `ACREEDOR`), se validan por **reglas de negocio del servicio**, no por el
  XSD.

---

## [2.3.0] — 2026-07-21

Cambio **MINOR** (aditivo y retrocompatible): se incorpora el acto de **Permuta**.
Mismo namespace y `version="2.0"`; compraventa, hipoteca y donación existentes siguen
validando sin cambios. Permuta es **estructuralmente idéntica a compraventa** (partes
con rol `TRANSMITENTE` y `ADQUIRENTE`; inmuebles y montos a nivel `<Acto>`) y **reutiliza
el bloque económico** de compraventa (`ValuacionFiscal` + `Monto`): no agrega estructura
económica propia.

Modelo de la permuta: una permuta de N inmuebles se representa como **UN testimonio con
N `<Acto>` de permuta** (uno por matrícula), agrupados por el mismo trámite y un solo PDF
firmado. Cada acto es **simple** —un transmitente y un adquirente—; el "cruce" característico
(una persona transmite en un acto y adquiere en otro) ocurre **entre actos**, nunca dentro
de uno. El **PRECIO** (`Monto`) se **repite igual en cada acto** —es el valor de la operación,
no se reparte—; la **VALUACIÓN FISCAL** (`ValuacionFiscal`) es la del inmueble de cada acto y
por eso difiere entre actos.

### Agregado
- `xsd/v2/actos/permuta.xsd`: tipo `PermutaType`. Mismo patrón que `CompraventaType`
  (textos libres opcionales del acto, incluido `AsentimientoConyugal`); el sustento vive
  a nivel `<Acto>`.
- `<xs:element name="Permuta" type="PermutaType"/>` **activado** en el `xs:choice` de
  `ActoType` (`testimonio-digital.xsd`) + include del nuevo XSD.
- Ejemplo `ejemplos/v2/permuta-ejemplo-valido.xml` (datos **ficticios**) con **dos actos**
  de permuta: el mismo `Monto` repetido en ambos y valuaciones fiscales distintas.

### Sin cambios
- `comunes/datos-economicos.xsd` **no se toca**: permuta usa `ValuacionFiscal`/`Monto`
  tal como están (opcionales a nivel XSD). Qué monto exige cada acto lo validan las reglas
  de negocio del servicio: permuta exige `ValuacionFiscal` + `Monto` (precio), igual que
  compraventa.
- Los roles `ADQUIRENTE`/`TRANSMITENTE` y su ruteo ya existían: permuta no agrega roles.

---

## [2.2.0] — 2026-07-16

Cambio **MINOR** (aditivo y retrocompatible): se incorpora el acto de **Donación**.
Mismo namespace y `version="2.0"`; compraventa e hipoteca existentes siguen validando
sin cambios. Donación es **estructuralmente idéntica a compraventa** (partes DONANTE =
`TRANSMITENTE`, DONATARIO/A = `ADQUIRENTE`; inmuebles y montos a nivel `<Acto>`) y
**reutiliza el bloque económico** de compraventa (el valor del acto va en
`DatosEconomicos/Monto`): no agrega estructura económica propia.

### Agregado
- `xsd/v2/actos/donacion.xsd`: tipo `DonacionType`. Mismo patrón que `CompraventaType`
  (textos libres opcionales del acto, incluido `AsentimientoConyugal`); el sustento vive
  a nivel `<Acto>`.
- `<xs:element name="Donacion" type="DonacionType"/>` **activado** en el `xs:choice` de
  `ActoType` (`testimonio-digital.xsd`) + include del nuevo XSD.
- Ejemplo `ejemplos/v2/donacion-ejemplo-valido.xml` (datos **ficticios**).

### Sin cambios
- `comunes/datos-economicos.xsd` **no se toca**: donación usa `ValuacionFiscal`/`Monto`
  tal como quedaron en 2.1.0 (opcionales). El `Monto` es el valor del acto de donación.

---

## [2.1.0] — 2026-07-14

Cambio **MINOR** (aditivo y retrocompatible): se incorpora el acto de **Hipoteca**.
Mismo namespace y `version="2.0"`; los testimonios de compraventa existentes siguen
validando sin cambios. Los roles `ACREEDOR`/`DEUDOR` (ya presentes desde 2.0.0) pasan
a estar **disponibles** con este release.

### Agregado
- `xsd/v2/actos/hipoteca.xsd`: tipo `HipotecaType`. Es el discriminador del acto en el
  `xs:choice`; el sustento (partes con rol `ACREEDOR`/`DEUDOR`, inmuebles, montos) vive a
  nivel `<Acto>`, igual que en compraventa. Sin campos propios por ahora.
- `<xs:element name="Hipoteca" type="HipotecaType"/>` **activado** en el `xs:choice` de
  `ActoType` (`testimonio-digital.xsd`) + include del nuevo XSD.
- `comunes/datos-economicos.xsd`: elemento `MontoHipoteca` (capital del gravamen:
  `valor` + `moneda` + `cotizacion`), del mismo tipo que `Monto`.
- Ejemplo `ejemplos/v2/hipoteca-ejemplo-valido.xml` (datos **ficticios**).

### Cambiado
- `comunes/datos-economicos.xsd`: `ValuacionFiscal` y `Monto` pasan a `minOccurs=0`
  (opcionales a nivel XSD). Es una **relajación retrocompatible**: los testimonios de
  compraventa que ya los incluyen siguen validando. Qué monto exige cada acto se valida por
  **reglas de negocio del servicio**, no por el XSD: compraventa exige `ValuacionFiscal` +
  `Monto` (precio); hipoteca exige `MontoHipoteca` y **no** lleva valuación ni precio de venta.

---

## [2.0.0] — 2026-06-24

Cambio **MAJOR**: un testimonio pasa de **un acto** a **N actos (1..N)**. Nuevo
namespace `https://contrato.rpi.jusneuquen.gov.ar/testimonio-digital/v2` y
atributo raíz `version="2.0"`. v2 **coexiste** con v1: los XSD de v1 quedan
intactos en `xsd/` y los de v2 viven en `xsd/v2/`.

### Agregado
- Contenedor `<Actos>` con `<Acto numero="N">` (`minOccurs=1`, sin tope). Actos
  de tipos heterogéneos permitidos. El atributo `numero` es entero positivo
  obligatorio (su unicidad la valida el servicio, no el XSD).
- `xsd/v2/comunes/parte.xsd`: tipo `ParteType` (= `PersonaType` + atributo `rol`)
  y enum `RolParteEnum` con `ADQUIRENTE`, `TRANSMITENTE`, `ACREEDOR`, `DEUDOR`
  (`ACREEDOR`/`DEUDOR` quedan listos para Hipoteca, aún no disponible).
- Ejemplo `ejemplos/v2/compraventa-dos-actos.xml`: testimonio con dos actos.
- Migración de los 6 ejemplos a `ejemplos/v2/`.
- Códigos de error de negocio: `NUMERO_ACTO_DUPLICADO`, `ROL_NO_VALIDO_PARA_ACTO`,
  `ACTO_SIN_ADQUIRENTE`.

### Cambiado
- **Bajan del nivel testimonio al nivel `<Acto>`**: `Partes` (antes
  `Adquirentes`/`Transmitentes`), `Inmuebles`, `DatosEconomicos`,
  `CertificacionCatastral`, `NomenclaturaCatastral` (opcional),
  `CertificacionRegistralPrevia` y `VisadoRentas`. Cada acto tiene su propio
  juego de estos bloques.
- **Partes con rol genérico**: se reemplazan los contenedores `Adquirentes` /
  `Transmitentes` de v1 por una lista de `<Parte rol="...">`. `CompraventaType`
  ya no contiene partes ni inmuebles: solo los campos propios del tipo
  (`DescripcionActoIncompleto` y los textos libres).
- `PROPORCIONES_NO_SUMAN_UNO`: la suma de proporciones de adquirentes se valida
  **por acto**.
- Documentación de `docs/` reescrita a v2.

### Quedan a nivel testimonio (sin cambios respecto de v1)
- `MetadatosEnvio`, `EscribanoAutorizante`, `Otorgamiento`, `Rogante`,
  `TextoCuerpo`, `Observaciones` y `<ds:Signature>` (un trámite, un cuerpo de
  escritura por testimonio).

---

## [2.0.0] — 2026-06-24 · detalle de los cambios

### Cambios mayores en el contrato (alineamiento al Excel del RPI)

#### Agregado
- Elemento `<TextoCuerpo>` obligatorio: transcripción del cuerpo del acto
  en texto plano (max 500.000 caracteres). Hijo directo de
  `<TestimonioDigital>`, ubicado después de `<Rogante>` y antes de
  `<Observaciones>`. Usado por el servicio del RPI para análisis NLP,
  visualización y búsqueda full-text.
- Soporte para personas jurídicas (Tipo=J) y organismos públicos (Tipo=O).
- Bloque `Representante` opcional en personas (adquirentes y transmitentes).
- Bloque `CertificacionCatastral` con campos condicionales.
- Bloque `NomenclaturaCatastral` opcional con 5 subcampos.
- Bloque `VisadoRentas` obligatorio.
- Campos en `<Compraventa>`: `DescripcionActoIncompleto`, `ReconocimientoHipotecaMedidasCautelares`, `AfectacionesAlDominio`, `AsentimientoConyugal`.
- Campos de contacto obligatorios en `Rogante`: `NumeroRegistro`, `Localidad`, `Provincia`, `Domicilio`, `Telefono`.
- Campos opcionales en `Persona` (humana): `Nupcias`, `Conyuge`, `InscripcionOrganismoSede`.
- En `IdentificacionInmueble`: campos `Barra`, `Tomo`, `Folio`, `Finca`.
- Tres ejemplos nuevos: `compraventa-persona-juridica.xml`, `compraventa-con-representante.xml`, `compraventa-inmueble-antiguo.xml`.
- Documento `docs/10-campos-del-formulario.md` con tabla plana de campos.

#### Cambiado
- `xsd/xmldsig-core-schema.xsd`: cambio de `namespace="##any"` a
  `namespace="##other"` en `SignatureMethodType`, `CanonicalizationMethodType`
  y `DigestMethodType`. Corrige violación de Unique Particle Attribution
  (UPA) que hace que el XSD canónico de W3C falle con validadores
  estrictos como xmllint.
- `xsd/testimonio-digital.xsd`: agregado atributo opcional `Id` (xs:ID)
  al elemento raíz `TestimonioDigital`. Necesario para compatibilidad
  con librerías de firma XML-DSig que agregan automáticamente este
  atributo (xml-crypto, Apache Santuario, etc.).
- `docs/05-firma-digital.md`: agregada sección sobre compatibilidad con
  firmadores que agregan `Id`. Agregada nota sobre el estado actual de
  verificación de certificado en v1.0.
- `IdentificacionInmueble/Matricula`: ahora es `xs:integer` 1-8 dígitos (antes string "DD-NNNN").
- `IdentificacionInmueble/Departamento`: ahora es código numérico 1-16 con enum (antes string libre).
- `IdentificacionInmueble`: campo `Barrio` renombrado a `Barra` (alineamiento al Excel).
- `CertificacionRegistralPrevia/FechaEmision` renombrada a `FechaEmisionPrimera`.
- `CertificacionRegistralPrevia/FechaVigencia` renombrada a `FechaEmisionSegunda`.
- `EscribanoAutorizante/Nombre`: longitud max ajustada a 60 (antes 120).
- `EscribanoAutorizante/RegistroNumero`: longitud max ajustada a 8 (antes 10).
- `EscribanoAutorizante/Sede`: longitud max ajustada a 40 (antes 60).
- `Persona/Proporcion`: ahora string fracción "N/D" (antes atributos numerador/denominador).
- Enum `TipoDocumento`: removido `CI` (alineamiento al Excel: solo DNI, LE, LC, PAS).
- Archivo `xsd/comunes/persona-humana.xsd` renombrado a `xsd/comunes/persona.xsd`.

#### Removido
- `Persona/Documento` con atributo `tipo`: reemplazado por `TipoDocumento` y `NumeroDocumento` como elementos separados (alineamiento al Excel).

---

Cambios pendientes antes de la primera publicación oficial:

### A definir antes de v1.0.0

- URL final del endpoint en staging y producción.
- Mecanismo de autenticación entre cliente y RPI (Bearer token vs mTLS).
- Mecanismo de autenticación de los callbacks (RPI → cliente).
- Requisitos específicos de la cadena de certificación para firma XML-DSig
  en el ámbito del Poder Judicial de Neuquén.
- Email de contacto para reportes de seguridad.

---

## [1.0.0-draft] — 2026-06-02

Borrador inicial del contrato. Se publica para revisión por parte del Colegio
de Escribanos de Neuquén y validación interna del RPI.

### Agregado

#### Contrato técnico

- Endpoint `POST /testimonios` con `multipart/form-data` (partes `xml` y `pdf`).
- Estructura XML del testimonio digital con secciones:
  - `MetadatosEnvio` (identificador, timestamp, hash PDF, versión).
  - `EscribanoAutorizante` (nombre, CUIT, registro, sede).
  - `Otorgamiento` (lugar, fecha, número de escritura, folio).
  - `CertificacionRegistralPrevia` (número, fechas).
  - `DatosEconomicos` (valuación fiscal, monto, moneda, cotización).
  - `Acto` con `Compraventa` (adquirentes, transmitentes, inmuebles).
  - `Rogante`.
  - `Observaciones` (opcional).
  - Firma XML-DSig embebida.

#### Firma digital

- Estándar XML-DSig enveloped con RSA-SHA256 y Canonical XML 1.0.
- Certificado X.509 incluido en `<KeyInfo>`.
- Verificación de coincidencia entre CUIT del certificado y del XML.

#### Adjunto PDF

- PDF firmado digitalmente (recomendado: PAdES).
- Verificación de integridad por hash SHA-256 declarado en el XML.

#### XSDs modulares

- `testimonio-digital.xsd` como entry point.
- 8 archivos comunes reutilizables entre actos en `xsd/comunes/`.
- `xsd/actos/compraventa.xsd` con el único acto soportado en v1.0.
- Inclusión local del XSD oficial de W3C XML-DSig para validación offline.

#### Documentación

- 9 documentos en `docs/` cubriendo introducción, flujo end-to-end, endpoint,
  XML, firma, PDF, errores, callbacks y glosario.
- README principal con índice navegable.
- 3 ejemplos XML válidos en `ejemplos/`.

#### Manejo de errores y reintentos

- Catálogo de códigos HTTP esperados (200, 202, 4xx, 5xx).
- Catálogo de códigos de error específicos del contrato.
- Política de reintentos con backoff exponencial para errores 5xx.
- Garantía de idempotencia por `IdentificadorEnvio` (UUID v4).

#### Callbacks

- Sistema de notificaciones del RPI al cliente con 7 tipos de eventos:
  - `validacion_completada`
  - `sincronizacion_completada`
  - `entrada_general_asignada`
  - `inscripcion_provisoria`
  - `inscripcion_definitiva`
  - `rechazo_registral`
  - `validacion_fallida`

#### Gobernanza

- `GOVERNANCE.md` con roles, procesos de cambio, política de versionado.
- `CONTRIBUTING.md` con guía de contribución.
- `SECURITY.md` con política de reporte de vulnerabilidades.
- `LICENSE.md` con licencia Creative Commons BY 4.0.

### Limitaciones conocidas de v1.0

- Solo se soporta el acto de **compraventa**.
- Solo se aceptan **personas humanas** como adquirentes y transmitentes.
- Sin representantes legales ni poderes.
- Sin asentimiento conyugal estructurado (puede aparecer en el cuerpo del
  testimonio en texto libre).
- Sin manejo avanzado de PEP (solo flag booleano).
- Sin múltiples actos en un mismo testimonio.

Estas limitaciones se abordarán en versiones futuras según las prioridades
del proyecto.

---

## Política de versionado

| Tipo | Formato | Cuándo se aplica | Ejemplo |
|------|---------|------------------|---------|
| MAJOR | `X.0.0` | Cambios incompatibles | Cambiar estructura raíz del XML |
| MINOR | `1.X.0` | Cambios compatibles que agregan funcionalidad | Agregar tipo de acto, campo opcional |
| PATCH | `1.0.X` | Correcciones que no cambian el contrato | Typos, ejemplos, aclaraciones de docs |

El namespace del XML incluye solo la versión MAJOR (`/v1`, `/v2`). Los
cambios MINOR y PATCH mantienen el mismo namespace. Los cambios MAJOR
cambian el namespace.

Ver [GOVERNANCE.md](GOVERNANCE.md) para el proceso completo de releases.
