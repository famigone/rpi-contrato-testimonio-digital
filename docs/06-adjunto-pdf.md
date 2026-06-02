# 06 — Adjunto PDF

## Qué es el PDF

El PDF que viaja junto al XML es el **testimonio en sí**: el documento legal
firmado por el escribano que constituye el testimonio de la escritura.

Funcionalmente equivale al testimonio en papel que hoy se presenta físicamente
al RPI. La diferencia es que viaja digitalmente y está firmado con firma
digital.

## Cómo viaja

El PDF se envía en la misma petición HTTP que el XML, como parte del
`multipart/form-data` (ver [03 — Endpoint API](03-endpoint-api.md)).

- **Nombre del campo**: `pdf`.
- **Content-Type**: `application/pdf`.

## Firma del PDF

El PDF debe estar firmado digitalmente por el escribano autorizante. El RPI
verifica que el PDF tenga firma digital válida.

**Formato recomendado**: PAdES (PDF Advanced Electronic Signatures), que es el
estándar europeo y el más usado en firma digital de PDFs en Argentina.

El certificado usado para firmar el PDF debería ser el mismo que firma el XML,
aunque el RPI no exige que coincidan estrictamente (algunos firmadores
generan certificados específicos para PDF). Lo importante es que el PDF tenga
firma válida verificable.

## Hash del PDF en el XML

Para garantizar la **integridad** del PDF y vincularlo inequívocamente al XML,
el XML incluye el hash SHA-256 del PDF en `MetadatosEnvio/HashPDF`.

El sistema del Colegio debe:

1. Generar el PDF firmado.
2. Calcular el hash SHA-256 del PDF (el archivo completo, byte a byte).
3. Codificar el hash en hexadecimal lowercase.
4. Incluir ese hash en el XML antes de firmar el XML.

El RPI:

1. Recibe el PDF y el XML.
2. Calcula el hash SHA-256 del PDF recibido.
3. Compara con el `HashPDF` declarado en el XML.
4. Si no coinciden, rechaza el envío con HTTP 400.

Esto garantiza que **el PDF que llega al RPI es exactamente el que el escribano
firmó** y que el XML no fue manipulado para apuntar a otro PDF.

## Cómo calcular el hash

Ejemplo en distintos lenguajes:

### Bash

```bash
sha256sum testimonio.pdf | awk '{print $1}'
```

### Python

```python
import hashlib
with open("testimonio.pdf", "rb") as f:
    hash_pdf = hashlib.sha256(f.read()).hexdigest()
```

### Java

```java
MessageDigest md = MessageDigest.getInstance("SHA-256");
byte[] hashBytes = md.digest(Files.readAllBytes(Paths.get("testimonio.pdf")));
String hashHex = new BigInteger(1, hashBytes).toString(16);
// padding a 64 caracteres si hace falta
```

### Node.js

```javascript
const crypto = require('crypto');
const fs = require('fs');
const hash = crypto.createHash('sha256');
hash.update(fs.readFileSync('testimonio.pdf'));
const hashHex = hash.digest('hex');
```

### .NET

```csharp
using (var sha = SHA256.Create()) {
    var bytes = File.ReadAllBytes("testimonio.pdf");
    var hashBytes = sha.ComputeHash(bytes);
    var hashHex = BitConverter.ToString(hashBytes).Replace("-", "").ToLower();
}
```

## Tamaño máximo

El PDF puede pesar hasta **45 MB** (el total de la petición no puede superar
50 MB contando XML, headers y boundaries del multipart).

Si el PDF es muy grande, conviene optimizarlo antes de enviarlo (eliminar
imágenes innecesarias, comprimir, etc.).

## Contenido del PDF

El PDF debe ser el testimonio completo. Conviene que incluya:

- Encabezado con identificación del registro notarial.
- Cuerpo de la escritura (cláusulas, identificación de partes, etc.).
- Asentimiento si corresponde.
- Firmas (digitales) del escribano y de las partes si así lo requiere la
  normativa.
- Sello digital del Colegio si aplica.

El RPI **no parsea el contenido del PDF** — confía en los datos estructurados
del XML. El PDF se almacena para fines legales y de auditoría, y se puede
recuperar para inspección humana cuando se necesita.

## Buenas prácticas

- **Firmar el PDF primero, calcular el hash, luego firmar el XML.** Si firmás el
  XML antes que el PDF, vas a tener que recalcular el hash y refirmar todo.
- **Validar el hash antes de enviar.** Recalcular el hash del PDF que vas a
  enviar y compararlo con el del XML evita rechazos del RPI.
- **No modificar el PDF después de firmar el XML.** Cualquier cambio (incluso
  agregar metadata) altera el hash y rompe la integridad.

## Próximos pasos

- Para códigos de error relacionados con PDF y hash, andá a
  [07 — Respuestas y errores](07-respuestas-y-errores.md).
- Para ejemplos de envíos completos, mirá `../ejemplos/`.
