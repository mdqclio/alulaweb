# Remediación — Alula Hostel

- **Repositorio:** `alulaweb` (`git@github.com:mdqclio/alulaweb.git`)
- **Fecha:** 2026-06-06
- **Base:** `docs/auditoria/alulaweb.md`
- **Alcance de esta tanda:** SOLO arreglos seguros y reversibles en código. NO se desplegó nada, NO se reescribió historial, NO se hizo force-push. La config de Firebase sigue en placeholders (`TU_API_KEY`, etc.).

**Leyenda:** ✅ hecho en código · 📄 documentado / preparado para que el dueño lo aplique · ⏳ pendiente (nice-to-have, no hecho ahora)

---

## ✅ Hecho en esta tanda

### ✅ Reglas Firebase deny-by-default (audit punto 2 y 4)
- **`database.rules.json`** — Realtime Database con `.read`/`.write` en `false` por defecto. Única excepción: crear nodos en `sugerencias_huespedes/pendientes/{id}` (solo create, no update/delete: `!data.exists() && newData.exists()`), con validación de forma, tipos y tamaño de cada campo (`nombre`, `categoria`, `lugar`, `porQue` obligatorios; `estado` forzado a `"pendiente"`; campos desconocidos rechazados con `$other: false`). Esto traslada a las reglas la validación que hoy es solo client-side y bypasseable.
- **`storage.rules`** — Storage deny-by-default. Única excepción: subir `sugerencias/{id}.jpg` validando `contentType image/*` y tamaño `<= 5 MB` (mismo límite que aplica el cliente). Lectura/listado públicos bloqueados (las URLs se obtienen vía `getDownloadURL` tras subir).
- **`firebase.json`** — apunta a ambos archivos de reglas (`database` + `storage`).
- **`.firebaserc`** — `default: alula-hostel` (projectId detectado en `index.html`).

> ⚠️ **Estas reglas NO se desplegaron.** Son archivos listos para `firebase deploy`. Ver "Pasos para el dueño" abajo. **Deben desplegarse ANTES de pegar la config real** en `index.html`, si no la DB queda abierta entre que se pega la config y se suben las reglas.

> ⚠️ **Nota importante sobre las lecturas de disponibilidad.** El consultor de disponibilidad lee desde el cliente `configuracion`, `tarifas/temporadas` y `reservas`. Con estas reglas deny-by-default **esas lecturas quedan bloqueadas** (el feature de disponibilidad dejará de funcionar hasta resolverlo). Esto es **intencional**: el audit (punto 2) marca que exponer `reservas` a cualquier visitante filtra datos de ocupación. Opciones para el dueño, en orden de preferencia:
> 1. **(Recomendado)** Mover el cálculo de disponibilidad a una **Cloud Function** que lea `reservas` server-side y devuelva solo disponible sí/no + precio. El cliente nunca lee `reservas`.
> 2. Si se acepta el riesgo a corto plazo, abrir lectura explícita SOLO de `configuracion` y `tarifas/temporadas` (datos no sensibles) y dejar `reservas` cerrado, ajustando el cliente para no depender de leer reservas crudas.
> 3. (No recomendado) Abrir `reservas` a `.read: true` — expone ocupación. Documentado solo para que sea una decisión consciente, no un default silencioso.

### ✅ Config de Firebase unificada (audit punto 5)
- Antes: `firebaseConfig` estaba **duplicada** en los dos `<script type="module">` (sugerencias y disponibilidad) → riesgo de desincronización.
- Ahora: una **única definición** en un `<script>` previo que setea `window.__ALULA_FIREBASE_CONFIG__`, referenciada por ambos módulos. Sin cambio de comportamiento (mismos valores, mismo orden de inicialización). Quedó **un solo lugar** donde pegar la config real.
- Verificado: `grep -c TU_API_KEY index.html` = 1 (antes 2).

### ✅ `.gitignore` (faltaba)
- Creado: ignora `node_modules/`, `.env*`, `*serviceAccount*.json`, `.firebase/`, logs de firebase-debug, basura de SO/editores. Refuerza la disciplina del audit punto 3 (nunca commitear claves de Admin SDK / service account).

---

## 📄 Verificación realizada

- `python3 json.load` sobre `database.rules.json`, `firebase.json`, `.firebaserc` → **JSON válido**.
- `node --check` sobre los 2 `<script type="module">` extraídos (sin las líneas `import` de URL) + el bloque `<script>` de config compartida → **sintaxis OK** en los tres.
- `grep -c TU_API_KEY index.html` = **1** (config deduplicada correctamente).

---

## 📄 / ⏳ Pendiente — dueño (Leo / Coni)

### 📄 Desplegar reglas + App Check ANTES de poner la config real (audit punto 1, 2, 7) — dueño
**Orden obligatorio:**
1. `firebase login` (cuenta con acceso al proyecto `alula-hostel`).
2. `firebase deploy --only database,storage` → sube `database.rules.json` y `storage.rules`.
3. Verificar en consola Firebase que las reglas activas son las del repo (no "modo prueba").
4. Resolver la lectura de disponibilidad (ver nota arriba: Cloud Function recomendada).
5. **Recién entonces** pegar la config real en `window.__ALULA_FIREBASE_CONFIG__` (único lugar) y commitear. La `apiKey` es pública (OK que viva en el cliente); **nunca** commitear claves de Admin SDK / service account.
6. Activar **Firebase App Check** (reCAPTCHA v3 / Enterprise) y enforcement en Realtime Database + Storage para frenar abuso de escrituras/lecturas (audit punto 7, rate-limiting real del lado server; el `localStorage` actual es cosmético).

### 📄 Monitoreo / analytics de los `catch` de Firebase (audit punto 10) — dueño
- Hoy los errores de Firebase se tragan en `console.error`/`console.warn` (`index.html`: "Error enviando sugerencia", catches del módulo de disponibilidad). Si el form UGC o Firebase fallan, nadie se entera y se pierden reservas en silencio.
- Sumar: analytics liviano (Plausible o GA4) + error-tracker (Sentry, o como mínimo enviar los `catch` a un endpoint/log). Reemplazar los `console.error` por reporte real.
- No se hizo en código en esta tanda porque requiere decidir proveedor y tokens/DSN (config que el dueño debe proveer); es un cambio de producto, no un arreglo reversible.

### ⏳ Optimización de imágenes (audit punto 8) — nice-to-have, NO hecho ahora
- `fotos/` ~4.6 MB; `dorm-4.jpg` y `living-arco.jpg` ~1.2 MB cada uno; JPG (no WebP/AVIF); hero 328 KB como `background-image`. Las `<img>` cargan eager y **sin `width`/`height`** → CLS.
- Pendiente (puede ser pesado, requiere reconvertir binarios): convertir a WebP/AVIF, agregar `width`/`height` explícitos y `loading="lazy"` a las `<img>` de habitaciones e histórica.
- Impacto: performance / Core Web Vitals / consumo de datos mobile. No afecta disponibilidad → no es bloqueante.

### ⏳ Otros nice-to-have del audit (no tocados) — dueño
- Tailwind por CDN (modo JIT en navegador, no apto para prod): considerar build con Tailwind CLI y CSS minificado.
- URLs absolutas a `mdqclio.github.io/alulaweb` en imágenes/hero: pasar a rutas relativas para desacoplar del hosting.
- `CNAME` / `.nojekyll` si se quiere dominio propio.

---

## Resumen de archivos tocados

| Archivo | Acción |
|---|---|
| `database.rules.json` | ✅ nuevo — RTDB deny-by-default + validación de sugerencias |
| `storage.rules` | ✅ nuevo — Storage deny-by-default + subida validada de fotos |
| `firebase.json` | ✅ nuevo — apunta a ambas reglas |
| `.firebaserc` | ✅ nuevo — projectId `alula-hostel` |
| `.gitignore` | ✅ nuevo |
| `index.html` | ✅ config Firebase unificada (`window.__ALULA_FIREBASE_CONFIG__`), sin cambio de comportamiento |
| `docs/auditoria/alulaweb-REMEDIACION.md` | 📄 este documento |
