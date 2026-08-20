# Cambios — rama `mejoras-web`

Ediciones de contenido sobre `index.html` (archivo único del sitio estático).
No se tocó: chequeador de disponibilidad ni su JS, formulario de la guía (UGC),
`firebase.json`, reglas de Firebase ni ningún otro archivo.

---

## 1. Título y meta description

- `<title>`: "Alula Hostel — Dormí en la historia de Mar del Plata"
  → **"Alula Hostel — Hostel frente al mar en La Perla, Mar del Plata"**
- `<meta name="description">` reemplazada por la nueva, que incluye casona de 1932,
  La Perla, precio desde $27.000, habitaciones para grupos y reserva por WhatsApp.

## 2. "La Perla" en lugar de "Loma de Santa Cecilia"

Reemplazado en:

- **Hero** — "Una joya de 1932 frente al mar en La Perla."
- **Sección Ubicación** — título "En *La Perla*, frente al mar."
- **Footer** — descripción de marca y línea de pie ("La Perla · Mar del Plata · AR").

Se conservó **una sola** mención patrimonial, dentro de la sección Historia:

> "En La Perla, en la Loma de Santa Cecilia, punto fundacional de la ciudad, se levanta
> una joya pintoresquista de 650 m²…"

## 3. Precio visible arriba del chequeador de disponibilidad

En la sección Habitaciones, justo encima del botón "Consultar disponibilidad", se agregó
un bloque enmarcado con el estilo existente (borde madera sobre fondo marino):

> **Camas desde $27.000 por noche** (estadías de 3+ noches)
> Menos de 3 noches: $30.000

## 4. Nueva sección "Vení en grupo" (`#grupos`)

Ubicada entre Habitaciones y Reseñas. Título "La casa para tu grupo", con el texto
pedido y botón de WhatsApp con mensaje precargado
"Hola! Quiero consultar por una habitación para grupo".

**Fotos:** en `/fotos` no hay imágenes de mesa de pool ni de jardín. Se usaron las
existentes que aplican (`living-arco.jpg` como living histórico, más `fachada.jpg` y
`dorm-vista-mar.jpg`) y quedaron **dos slots comentados con `TODO`** en el HTML para
`fotos/pool.jpg` y `fotos/jardin.jpg`: al subir esas fotos, basta con descomentar.

## 5. Nueva sección FAQ (`#faq`)

Ubicada inmediatamente antes de la sección de reserva (`#reservar`), con los 6 pares
pregunta/respuesta exactos indicados: precio por persona, no hay privadas/matrimoniales
(sí habitación exclusiva desde 4 personas), no se aceptan mascotas, hostel para adultos,
cómo reservar por WhatsApp y dirección.

## 6. Público objetivo (sección Historia)

- "pensada para nómadas digitales, viajeros lentos y quienes saben que dormir bajo
  techos así no se olvida"
  → **"pensada para viajeros que buscan algo más que una cama: una casa con historia
  frente al mar"**

---

## Verificaciones

- **WhatsApp:** los 6 links `wa.me` del sitio apuntan a `5492234397923` (incluidos los
  dos generados por JS en el chequeador y el botón flotante). Sin excepciones.
- **HTML:** estructura validada — no hay etiquetas sin cerrar ni cierres huérfanos.
  Servido con `python3 -m http.server` y verificado por HTTP 200 con todo el contenido
  nuevo presente.
- **Alcance del diff:** 94 líneas agregadas, 10 eliminadas, todas en `index.html`.
  Ninguna toca el JS del chequeador, el formulario de la guía ni la config de Firebase.

## Punto pendiente para decidir

Queda **una** mención a "Santa Cecilia" fuera de la sección Historia: dentro de una
**reseña textual de un huésped** en la sección Reseñas —

> "La casa es un sueño, la decoración es hermosa y la ubicación en la loma de Santa
> Cecilia es la más linda de Mardel…"

No se modificó porque es una cita literal de un huésped y editarla la desvirtúa.
Si querés sacarla, las opciones son reemplazar esa reseña por otra o recortar la cita
(por ejemplo, cerrando en "la decoración es hermosa"). Avisame y lo aplico.

## Sugerencia no aplicada (fuera de alcance)

El menú de navegación no incluye las secciones nuevas (Grupos, FAQ). Si querés que
aparezcan, lo agrego en una edición aparte.
