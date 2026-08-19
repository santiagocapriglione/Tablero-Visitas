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

---

# Base de Contactos (pestaña "Contactos" del mismo tablero)

Red de contactos para hacer negocios: gente a la que hay que volver a llamar cada tanto, con qué le interesa y de qué hablamos la última vez. Vive en el mismo `data.json`, array `contactos`, y se ve en la pestaña **Contactos** del tablero.

## Cuando Santiago manda un contacto nuevo (audio o texto, suele mandar varios juntos)

1. Leer SOLO `data.json`.
2. Por cada persona mencionada, crear un contacto (id: `c` + nombre sin espacios + ddmmaa) con lo que haya: nombre, cómo lo conoce (`relacion`), qué busca (`notas`), `tipos` de inversión y `temas`.
3. Si ya existe, NO duplicar: actualizar `notas`/`tipos` y agregar entrada a `interacciones`.
4. Si acaba de cruzarse o hablar con la persona → `ultimoContacto` = hoy y `proximoContacto` = hoy + `frecuenciaDias`. Si solo se acordó de alguien (no habló) → `ultimoContacto` vacío y `proximoContacto` = hoy (aparece como "a reactivar").
5. `frecuenciaDias` según calor del vínculo: 30 (caliente / negocio en curso), 45 (busca activamente), 60 (default), 90 (frío).
6. `prioridad`: `alta` si busca algo hoy o da negocio, `media` default, `baja` conocidos lejanos.
7. Teléfono en formato `+549...` para que funcione el botón de WhatsApp. Si no lo dijo, dejarlo vacío y anotarlo como pendiente en las notas (no salir a buscarlo salvo que lo pida).
8. Actualizar `updatedAt` del contacto y del root, commitear y pushear.
9. Confirmar en 2-3 líneas y listar quién quedó para reactivar.

## Cuando avisa que habló / tomó un café con alguien

`ultimoContacto` = fecha, `proximoContacto` = fecha + `frecuenciaDias`, agregar entrada a `interacciones` (`fecha`, `canal`: encuentro/whatsapp/llamada/café/mail, `nota` con lo hablado y lo que quedó pendiente). Si de ahí sale una operación concreta → sugerir pasarla al Tablero de Negociaciones; si sale una visita → al array `visitas`.

## Repaso cada 15 días

Hay evento recurrente en Google Calendar. En cada repaso: listar vencidos y los de la quincena, proponer a quién escribirle primero (prioridad alta + más días sin contacto), y pedirle a Santiago que sume los contactos nuevos que se le hayan cruzado. El botón de WhatsApp abre el chat con un saludo armado — **el mensaje lo manda siempre Santiago**, nunca automatizar el envío.

## Schema de contacto

```json
{
 "id": "c...",
 "nombre": "",
 "telefono": "+549...",
 "email": "",
 "relacion": "cómo lo conoce",
 "tipos": ["flipping","terrenos","desarrollos","pozo","construccion","renta","compraventa","oportunidades","captaciones","negocios"],
 "prioridad": "alta|media|baja",
 "temas": ["temas de interés"],
 "notas": "",
 "ultimoContacto": "YYYY-MM-DD",
 "frecuenciaDias": 60,
 "proximoContacto": "YYYY-MM-DD",
 "interacciones": [{ "fecha": "YYYY-MM-DD", "canal": "", "nota": "" }],
 "updatedAt": "ISO"
}
```

`tipos` es la lista cerrada que el tablero sabe mostrar (si hace falta una nueva categoría, agregarla también a `TIPOS` en index.html). El tablero calcula solo los días sin contacto y quién toca reactivar → costo cero de tokens.
