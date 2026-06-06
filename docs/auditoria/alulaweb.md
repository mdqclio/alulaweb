# Auditoría de readiness para producción — Alula Hostel

- **Repositorio:** `alulaweb` (`git@github.com:mdqclio/alulaweb.git`)
- **Fecha de auditoría:** 2026-06-06
- **Stack:** Sitio web estático de marketing. Un único `index.html` (~68 KB) + carpeta `fotos/` (~4.6 MB). Tailwind vía CDN (`cdn.tailwindcss.com`), Google Fonts, e integración cliente con **Firebase** (Realtime Database + Storage) para dos features: "Sumar lugar favorito" (UGC) y "Consultar disponibilidad". Hosting presumiblemente **GitHub Pages** (las URLs de imágenes y del hero apuntan a `https://mdqclio.github.io/alulaweb/...`). Sin backend propio.

**Leyenda:** 🟢 ok · 🟡 mejorable · 🔴 bloqueante · ⚪ N/A

---

## 1) Front comprimido / sin secretos en cliente — 🟡 mejorable

El HTML es razonablemente liviano, pero **no hay build ni minificación**: Tailwind se carga por CDN (modo JIT en navegador, no apto para producción según el propio Tailwind), y el CSS/JS va inline sin comprimir. **No hay secretos reales expuestos**: la config de Firebase está en placeholders (`apiKey: "TU_API_KEY"`, `messagingSenderId: "TU_SENDER_ID"`, `appId: "TU_APP_ID"`). Importante: la `apiKey` de Firebase **no es un secreto** — es un identificador público de proyecto y es correcto que viva en el cliente; la seguridad real depende de las Security Rules (ver punto 2).

> **Riesgo:** Bajo hoy. Tailwind CDN agrega latencia y un warning en consola; cuando se complete la config de Firebase, la `apiKey` será pública (esperado) pero la DB quedará expuesta sin reglas.

## 2) RLS / seguridad de base de datos — 🔴 bloqueante (cuando se active Firebase)

No es Postgres/RLS, pero el equivalente aplica: **Firebase Realtime Database con Security Rules**. El cliente escribe directo en `sugerencias_huespedes/pendientes` y sube fotos a Storage (`sugerencias/{id}.jpg`), y **lee** `configuracion`, `tarifas/temporadas` y `reservas`. No hay forma de verificar las reglas desde el repo (viven en la consola de Firebase, no versionadas). Si la DB queda en modo de prueba / abierta, **cualquiera puede leer todas las reservas (datos de ocupación), escribir basura ilimitada y llenar Storage**. La lectura de `reservas` desde el cliente expone fechas/camas/estado de reservas a cualquier visitante.

> **Riesgo:** Alto. Sin reglas estrictas: exfiltración de datos de reservas, spam masivo en la guía, abuso de Storage (costos). Hoy mitigado solo porque Firebase aún no está configurado.

## 3) Git sin secretos — 🟢 ok

Escaneo de **todo el historial** (`git log -p --all` filtrando `apiKey|secret|password|token|AIza|PRIVATE`, etc.): **cero hallazgos reales**. Los commits son subidas de fotos y ediciones de texto. La config de Firebase nunca se commiteó con valores reales (siempre placeholders). No hay `.env`, claves ni `node_modules`.

> **Riesgo:** Nulo. Mantener la disciplina al pegar la config real (la `apiKey` es OK; nunca commitear claves de Admin SDK / service account).

## 4) APIs con auth / validación — 🟡 mejorable

⚪ N/A en cuanto a APIs propias (no hay backend). Pero el sitio escribe a Firebase desde el cliente sin autenticación. Validación: **solo client-side** (`required`, `maxlength`, honeypot anti-bot, chequeo de campos obligatorios, límite de 5 MB y compresión de imagen a 1200px). Toda validación client-side es **bypasseable** — un atacante puede escribir directo a la DB saltando el formulario. La defensa real debe estar en las Security Rules (validación de forma/tamaño/tipos en reglas), que no se pueden auditar acá.

> **Riesgo:** Medio. Sin validación en reglas, el formulario UGC es un vector de inyección de datos arbitrarios.

## 5) Hosting / entornos / env vars — 🟡 mejorable

Hosting estático (GitHub Pages, inferido por las URLs absolutas `mdqclio.github.io/alulaweb`). **No hay separación de entornos** (dev/staging/prod) ni manejo de variables: la config de Firebase está hardcodeada inline y, peor, **duplicada** en dos `<script type="module">` (módulo de sugerencias y módulo de disponibilidad) — hay que mantenerla sincronizada a mano. No hay `CNAME` (sin dominio propio configurado en repo) ni `.nojekyll`. Las imágenes se referencian por URL absoluta a github.io en vez de rutas relativas, lo que acopla el HTML al hosting actual.

> **Riesgo:** Medio. Config duplicada = riesgo de desincronización. URLs absolutas dificultan mover de hosting o usar dominio propio.

## 6) Login / sesiones / vulnerabilidades — ⚪ N/A — sitio estático sin login

No hay login, sesiones ni área privada para el visitante. La moderación ("Coni revisa") ocurre fuera del sitio (consola de Firebase). Nota menor de seguridad front: se embeben iframes de terceros (Instagram reel, Google Maps) — riesgo bajo, son fuentes confiables y los enlaces externos usan `rel="noopener"` correctamente.

> **Riesgo:** N/A.

## 7) Rate limiting — 🟡 mejorable

El formulario UGC tiene un rate-limit **cosmético**: `localStorage` ("una sugerencia cada 12hs por device"). Es trivial de evadir (borrar localStorage, incógnito, otro navegador). No hay rate-limit del lado servidor. La consulta de disponibilidad no tiene límite y lee `reservas` completo en cada submit. El único freno real posible es vía Firebase (reglas + App Check), no presente/verificable.

> **Riesgo:** Medio. Spam y abuso de escrituras/lecturas posibles. Recomendado activar Firebase App Check.

## 8) Caché — 🟡 mejorable

⚪ Parcial: GitHub Pages sirve con CDN/caché razonable por defecto. Pero el sitio no controla headers de caché. **Las imágenes no están optimizadas y no tienen `width`/`height`** (solo el iframe del mapa usa `loading="lazy"`; las `<img>` de habitaciones y la histórica cargan eager). Total `fotos/` ~4.6 MB con archivos de **1.2 MB (`dorm-4.jpg`, `living-arco.jpg`)** sin formato moderno (JPG, no WebP/AVIF). El hero (328 KB) se carga como `background-image`. Falta de dimensiones explícitas en `<img>` provoca CLS (layout shift).

> **Riesgo:** Bajo-Medio. Impacto en performance/Core Web Vitals y consumo de datos en mobile, no en disponibilidad.

## 9) Escalabilidad — 🟢 ok

Sitio estático servido por CDN: escala prácticamente infinito en lecturas de HTML/imágenes. El punto de presión es Firebase (lectura de `reservas` completa por cada consulta de disponibilidad: O(n) sin paginación), pero a la escala de un hostel de 4 habitaciones es despreciable.

> **Riesgo:** Bajo.

## 10) Monitoreo / alertas — 🔴 bloqueante (operativo)

**No hay nada**: ni analytics (GA / Plausible), ni error tracking (Sentry), ni uptime, ni alertas. Los errores de Firebase se tragan en `console.error`/`console.warn` sin reportarse. Si Firebase falla o se llena la cuota, nadie se entera; si el formulario UGC deja de funcionar, no hay señal. Para un sitio que es el canal de captación de reservas (vía WhatsApp), no tener visibilidad de fallos ni de tráfico es una carencia real.

> **Riesgo:** Medio-Alto operativo. Caídas silenciosas del canal de reservas.

---

## Tabla resumen

| # | Punto | Estado |
|---|-------|--------|
| 1 | Front comprimido / sin secretos cliente | 🟡 |
| 2 | RLS / seguridad de base de datos (Firebase Rules) | 🔴 |
| 3 | Git sin secretos | 🟢 |
| 4 | APIs auth / validación | 🟡 |
| 5 | Hosting / entornos / env vars | 🟡 |
| 6 | Login / sesiones / vulns | ⚪ N/A |
| 7 | Rate limiting | 🟡 |
| 8 | Caché / optimización front | 🟡 |
| 9 | Escalabilidad | 🟢 |
| 10 | Monitoreo / alertas | 🔴 |

---

## Los 3 arreglos más urgentes

1. **Firebase Security Rules estrictas antes de poner la config real (punto 2).** Es el único bloqueante de seguridad. Reglas: `reservas`/`configuracion`/`tarifas` solo lectura de lo mínimo necesario (idealmente nada sensible legible desde el cliente — mover el cálculo de disponibilidad a una Cloud Function), y escritura a `sugerencias_huespedes/pendientes` validada por forma/tamaño/tipos + límite. Activar **Firebase App Check** para frenar abuso. Hoy está latente solo porque la config sigue en placeholders.

2. **Monitoreo mínimo del canal de reservas (punto 10).** Sumar analytics liviano (Plausible/GA) y un error-tracker (Sentry o al menos reportar los `catch` de Firebase). Sin esto, una caída del formulario o de Firebase pasa inadvertida y se pierden reservas en silencio.

3. **Unificar y parametrizar la config de Firebase + optimizar imágenes (puntos 5 y 8).** Dejar la config en un solo módulo compartido (hoy está duplicada en dos `<script>` y puede desincronizarse). En paralelo, convertir las fotos a WebP/AVIF, agregar `width`/`height` y `loading="lazy"` a las `<img>` para bajar de ~4.6 MB y eliminar CLS.

> **Nota:** El sitio es un marketing estático sano en lo estructural (git limpio, sin secretos, escala bien). Los riesgos de seguridad/operación son condicionales a que se complete la integración con Firebase — conviene resolver el punto 2 **en el mismo momento** en que se pegue la config real, no después.
