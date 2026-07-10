# Tablero de Visitas Houghton — Protocolo para Claude

App estática (index.html + data.json) hosteada en GitHub Pages, mismo patrón que el Tablero de Negociaciones:
- Repo: `santiagocapriglione/Tablero-Visitas` · URL: https://santiagocapriglione.github.io/Tablero-Visitas/
- En esta Mac hay `gh` autenticado en `~/bin/gh` (cuenta santiagocapriglione). Tras actualizar `data.json`, commitear y pushear a `main` — Pages republica solo en 1-2 min. **La fuente de verdad es `data.json` de esta carpeta.** El navegador lo mezcla con lo editado a mano en el celular (gana el `updatedAt` más nuevo por visita).

## Cuando Santiago manda un audio/mensaje con el feedback de una visita

El audio llega por la app de Claude del celular (o texto). Flujo, gastando mínimos tokens:

1. Leer SOLO `data.json` (no hace falta leer index.html).
2. Extraer del audio: **propiedad** (dirección/localidad), **contacto** (nombre, y teléfono si lo dice), **comentario** de la visita, **resultado** (positiva / neutra / negativa — inferirlo del tono si no lo dice explícito) y **calificación 1-5** si la visita fue positiva (si no la dice, pedírsela o estimar 3-4 y avisarle).
3. Crear la visita nueva (id `v` + timestamp base36) o actualizar una existente si es seguimiento de la misma visita (en ese caso agregar entrada a `seguimiento`, no duplicar la visita).
4. Completar los links de la propiedad (best effort, sin navegar de más):
   - `linkTokko`: si ya aparece en otra visita u otra app (Tablero de Negociaciones, planillas), reusarlo; si no, dejarlo vacío y completarlo la próxima vez que se navegue Tokko.
   - `linkHoughton`: link de la ficha en pablohoughton.com.ar si se conoce.
   - `dropbox`: ruta convención `10 Propiedades Generales / {Localidad} / {Dirección}` (Dropbox se consulta siempre vía web/navegador, no hay carpeta local).
   - `telefono` del contacto: si no lo dijo, buscarlo en WhatsApp/Tokko solo si Santiago lo pide o si hace falta para el recordatorio; el botón WhatsApp del tablero lo necesita en formato internacional `+549...`.
5. **Si el resultado es positivo → recordatorio a 48hs:**
   - `recontactar` = fecha de la visita + 2 días, `recontactado` = false.
   - Crear evento en Google Calendar (MCP de Calendar): título `Recontactar {nombre} — {dirección}`, fecha `recontactar` a las 10:00 (30 min), con recordatorio popup, y en la descripción el link `https://wa.me/{telefono sin +}?text={saludo breve mencionando la propiedad}` + el comentario de la visita. Guardar el id del evento en `calendarEventId`.
   - El tablero además muestra el panel "Recontactos pendientes" con botón de WhatsApp a un tap — el mensaje lo manda SIEMPRE Santiago desde su WhatsApp (nunca automatizar el envío: riesgo de ban y la negociación es 100% manual de Santiago).
6. Actualizar `updatedAt` de la visita y del root (ISO completo).
7. Commitear y pushear para que el tablero online se actualice.
8. Confirmarle en 2-3 líneas qué se registró y, si quedaron recontactos vencidos o para hoy/mañana, listárselos al final ("Ojo: tenés que recontactar a…").

## Cuando Santiago avisa que ya recontactó a alguien

Marcar `recontactado: true`, agregar entrada a `seguimiento` con lo conversado, actualizar `updatedAt`, pushear. Si de la charla sale una negociación (oferta), sugerirle pasarla al Tablero de Negociaciones — no duplicar acá.

## Chequeo de recontactos (mínimo costo de tokens)

- La app calcula y muestra sola los recontactos pendientes/vencidos → costo cero.
- Claude NO necesita monitoreo programado: en cada interacción sobre este tablero, tras cargar `data.json`, mencionar los recontactos de hoy y los vencidos.
- El recordatorio proactivo lo da Google Calendar (evento creado en el paso 5), que le suena en el celular sin gastar tokens.

## Schema de visita

```json
{
 "id": "v...",
 "fechaVisita": "YYYY-MM-DD",
 "propiedad": { "direccion": "", "localidad": "", "linkHoughton": "", "linkTokko": "", "dropbox": "" },
 "contacto": { "nombre": "", "telefono": "+549..." },
 "comentario": "",
 "resultado": "positiva|neutra|negativa",
 "calificacion": 0,
 "recontactar": "YYYY-MM-DD",
 "recontactado": false,
 "calendarEventId": "",
 "seguimiento": [{ "fecha": "YYYY-MM-DD", "nota": "" }],
 "updatedAt": "ISO"
}
```

`calificacion` solo aplica a visitas positivas (1-5). `recontactar` vacío en neutras/negativas salvo que Santiago pida seguimiento igual.
