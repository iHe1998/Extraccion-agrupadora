# Herramientas CyPe · Andreani

Herramienta interna de Joel Nuñez para el equipo de CyPe (Cuentas y Pedidos
Especiales) / Planificación en Andreani — operación farmacéutica.

Publicada en GitHub Pages: `https://ihe1998.github.io/Extraccion-agrupadora/`
Se comparte con los compañeros por WhatsApp y se instala como PWA en el celular.

---

## Archivos

| Archivo | Rol |
|---|---|
| `index.html` | **Toda la app.** SPA con dos secciones + modales. |
| `sw.js` | Service worker: caché offline y aviso de nueva versión. |
| `manifest.json` | Config PWA (íconos, colores, standalone). |
| `lector-etiquetas.html` | Solo redirige a `index.html#series`. Existe para no romper links viejos que ya circularon por WhatsApp. **No borrar.** |
| `logo.svg`, `icon-192.png`, `icon-512.png` | Marca. |

---

## Las dos herramientas

### 1. Extractor Agrupadora (`#agrupadora`)
Extrae el valor entre `<AA>` y `<AA>` de códigos SCE. Entrada pegada a mano o
por cámara (jsQR). Varios resultados se copian juntos separados por ` O `.

### 2. Lector de N° Serie (`#series`)
Escanea DataMatrix GS1 de cajas de medicamentos y extrae el campo `(21)`
(número de serie). Usa ZXing. Agrupa por laboratorio y permite copiar por grupo.

---

## Decisiones importantes (no revertir sin pensarlo)

### ZXing, no jsQR, para las series
jsQR **no lee DataMatrix**, solo QR. Las cajas de medicamentos usan DataMatrix.
El extractor de agrupadora sí puede seguir con jsQR porque esos son QR.

### Doble intento normal + invertido
Muchas etiquetas farmacéuticas son **blancas sobre fondo negro**. El scan
intenta primero normal y, si falla, con `luminance.invert()`. Forzar solo
invertido rompe la lectura de los códigos normales — ya pasó.

### Configuración por producto (lo más importante del proyecto)
El campo `(21)` de GS1 es de **longitud variable y sin separador confiable**.
ZXing a veces devuelve el separador GS (`\x1D`) y a veces lo pierde
completamente. Además el orden de los campos cambia entre laboratorios
(a veces `21` va después del `01`, a veces después del `17`, a veces al final).

Intentamos varias heurísticas (cortar en el próximo AI conocido, asumir 14
dígitos fijos) y **todas fallaron** con algún producto: hay series que
contienen `10` o `17` en el medio y el parser cortaba mal.

Solución actual: cada producto se configura **una vez** guardando la posición
exacta (`start` + `length`) del campo (21) dentro del código. El admin escanea
una unidad, escribe la serie que ve impresa en la caja, y el sistema encuentra
solo dónde está. El formato es fijo por producto, así que sirve para siempre.

- Producto configurado → lectura verificada (verde ✓)
- Producto nuevo → cae al fallback heurístico, marcado en ámbar (⚠) con botón
  para configurarlo en el momento

**Nunca confiar solo en la heurística para producción.** Es un respaldo.

---

## Firebase

Proyecto: `andreani-app-agrupadora-series` (plan Blaze, necesario para TTL).

### Colecciones
- **`productConfigs/{gtin}`** — permanente. `{ start, length, name, lab, updatedAt, updatedBy }`
- **`lecturas/{id}`** — efímera, TTL 8 horas vía campo `expiresAt`.

`length` puede ser un número o el string `'end'` (cuando la serie llega al
final del código).

### Auth
Login con Google **por popup**, no redirect. `signInWithRedirect` está roto en
Chrome 115+ cuando el sitio y el authDomain son dominios distintos (acá:
`github.io` vs `firebaseapp.com`) por el bloqueo de cookies de terceros. El
usuario se logueaba y volvía sin sesión.

Admin único: `joelihe1998@gmail.com` (constante `ADMIN_EMAIL`).

### Reglas de Firestore

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /productConfigs/{gtin} {
      allow read: if true;
      allow write: if request.auth != null
                   && request.auth.token.email == 'joelihe1998@gmail.com';
    }
    match /lecturas/{id} {
      allow read: if true;
      allow create: if true;
      allow update, delete: if request.auth != null
                            && request.auth.token.email == 'joelihe1998@gmail.com';
    }
  }
}
```

`allow create: if true` en `lecturas` es intencional: los compañeros no se
loguean y necesitan poder registrar sus escaneos. Trade-off aceptado para una
herramienta interna.

### Nota conocida
El login falla en el WiFi corporativo de Andreani (`ERR_CERT_AUTHORITY_INVALID`,
inspección SSL del proxy). Workaround: loguearse con datos móviles. No afecta a
los usuarios que solo leen.

---

## Laboratorios

```js
['ABBVIE','ASTRAZENECA','AMGEN','ROCHE','BIOSIDUS','SANOFI AVENTIS']
```

Se asigna al producto al configurarlo (fijo por producto, no por escaneo).

---

## Diseño

Rojo Andreani oficial: **`#d71920`** (tomado de andreani.com).

La app fue **unificada en un solo archivo** justo por un tema de diseño: cuando
eran dos HTML separados, el header "saltaba" al navegar entre ellos porque eran
dos copias del mismo CSS que se desincronizaban. Con SPA el header nunca se
redibuja. **No volver a separar en archivos por página.**

Detalles que costaron ajustar:
- `header` con `height: 64px` fijo (no `min-height`) y títulos con `nowrap` +
  ellipsis — si el texto se parte en dos líneas, el header crece y salta.
- `header img` necesita `width` **y** `height` fijos, o el logo se corre.
- El estado de la cámara es **un solo indicador** dentro del área de video.
  Antes había un texto con `bottom: -28px` que se salía del contenedor y se
  veía cortado.

---

## Flujo de actualización

1. Editar `index.html`
2. Subir el número de `CACHE` en `sw.js` (`agrupadora-v11` → `v12`)
3. Actualizar el `v11` del footer para que coincida
4. Commit + push

Los usuarios ven una barra "Nueva versión disponible" y actualizan con un toque.
El `sw.js` **no** debe llamar `skipWaiting()` en el `install` — eso es lo que
permite que la versión nueva espere la confirmación del usuario en vez de
activarse a mitad de uso.

---

## Preferencias de trabajo

- Joel prefiere **fragmentos puntuales para pegar** antes que regenerar el
  archivo completo, salvo que el cambio sea grande.
- Suele editar desde el celular vía la interfaz web de GitHub.
- Subir todos los archivos en **un solo commit** — commits separados disparan
  deployments que se cancelan entre sí en GitHub Pages.
