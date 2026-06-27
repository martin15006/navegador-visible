---
name: navegador-visible
description: >-
  Automatización de navegador VISIBLE en una ventana real, narrada paso a paso. Úsala SIEMPRE
  que el usuario quiera abrir webs, llenar formularios, hacer clicks, navegar, extraer datos
  o testear una app Y quiera VER en vivo lo que pasa — o cuando diga "mostrame", "que yo lo vea",
  "paso a paso", "en vivo", "abrí una ventana", "reproducí", "buscá en tal web", "no trabajes por detrás".
  Envuelve a agent-browser forzando el modo --headed (abre una ventana de Chrome REAL en el escritorio
  que el usuario mira en vivo, no headless), narra en español cada acción, verifica que cada acción
  importante haya funcionado de verdad, y pide CONFIRMACIÓN con botones antes de cualquier acción
  irreversible (enviar formularios, comprar, borrar, mandar mensajes). Prioriza esta skill por sobre
  la automatización de navegador por defecto cuando la transparencia importa.
license: MIT
---

# Navegador visible

Esta skill no reemplaza a `agent-browser` — lo **envuelve**. Usa sus comandos, pero
agrega una regla de comportamiento por encima: **el usuario tiene que poder ver y entender
cada cosa que hace el navegador, en el momento en que la hace, en una ventana real.**

El sentido es de confianza y aprendizaje. El usuario no quiere que el navegador trabaje
oculto y le devuelva un resultado mágico; quiere mirar por encima del hombro, ver el click,
ver el dato entrando en la tabla, y poder decir "pará" si algo va mal. Tratá cada sesión
como si le estuvieras **enseñando** lo que hacés mientras lo hacés.

> Dato clave: por defecto agent-browser corre **headless** (sin ventana, invisible). Esta skill
> existe justamente para forzar lo contrario. Si la sesión no se ve, la skill falló su propósito.

## Cómo se ve vs cómo navego (importante)

Hay dos cosas distintas, no las confundas:

- **La vista del usuario = la VENTANA real.** El usuario sigue todo mirando la ventana de Chrome
  en su escritorio. Esa es su pantalla, en vivo y a tamaño completo.
- **Mi forma de navegar = TEXTO.** Yo "leo" la página con `agent-browser snapshot -i`, que me
  devuelve los elementos como texto (botones, campos, links con sus refs `@eN`). Con eso me alcanza
  para saber qué tocar. **No necesito imágenes para trabajar.**

Por eso, **por defecto NO mando capturas de pantalla al chat.** Las imágenes en el chat son
redundantes (el usuario ya ve la ventana) y solo alargan la conversación. Mantené el chat limpio:
narrá en texto, navegá por `snapshot`, y dejá que la ventana sea el espectáculo.

**Sacá una captura solo si:**
1. Los elementos no se distinguen por texto y necesitás desambiguar (ej. dos resultados con título
   casi igual). Ahí guardá **una sola** captura a un archivo y miratela vos, o preguntale al usuario
   cuál es — no spamees el chat.
2. El usuario la pide explícitamente como registro.

Nunca saques una captura por cada paso. Si el usuario quiere un registro visual completo, ofrecé el
**video** (un solo archivo `.webm`) o el **dashboard** (ver "Opcionales"), no una lluvia de imágenes.

## Las 4 reglas de oro

### 1. Siempre ventana real (`--headed`)

Abrí el navegador en modo `--headed` para que aparezca una **ventana de Chrome de verdad en el
escritorio**. Nunca uses headless con esta skill. Pasá `--headed` en el primer comando que abre el
navegador (`agent-browser --headed open <url>`); la ventana queda abierta para los comandos siguientes.

### 2. Narrá liviano ANTES de actuar (en español)

Antes de cada acción importante, decí en **una línea** qué vas a hacer y por qué, para que el
usuario siga el hilo mientras mira la ventana. Ejemplo: *"Escribo la canción en el buscador."* /
*"Hago click en el primer resultado."*

Mantenelo liviano: como el usuario ya ve la ventana en vivo, no describas cada micro-movimiento ni
satures el chat. Narrá los pasos que importan, no cada tecla. Si pide más o menos detalle, ajustá.

### 3. Verificá que cada acción importante FUNCIONÓ (no la asumas)

Después de una acción cuyo éxito **no se ve obvio**, comprobá el resultado de verdad en vez de
suponer que salió. Esto evita decir "listo" cuando en realidad no pasó nada. Usá una verificación
por texto (no hace falta imagen):

- **Reproducir un video/audio:** chequeá que de verdad esté sonando (y no en pausa o muteado):
  ```bash
  agent-browser eval "(() => { const v = document.querySelector('video'); return v ? JSON.stringify({paused:v.paused, muted:v.muted, t:Math.round(v.currentTime)}) : 'no-video'; })()"
  ```
- **Login / navegación:** esperá la señal de éxito, ej. `agent-browser wait --url "**/dashboard"`
  o `agent-browser wait --text "Bienvenido"`.
- **Guardar / enviar:** esperá el mensaje de confirmación con `wait --text "Guardado"` o similar.

Si la verificación falla, decílo y corregí — no anuncies éxito sin pruebas.

### 4. Confirmá con botones ANTES de lo irreversible

Como el usuario está mirando y casi siempre son **sus cuentas reales**, frená antes de cualquier
acción difícil de deshacer: **enviar un formulario, comprar, borrar, mandar un mensaje, confirmar
un pago, aceptar términos.**

El procedimiento:
1. Describí en palabras **qué** está por pasar (qué datos, qué botón). El usuario lo ve en la ventana;
   si ayuda, podés guardar una captura a un archivo, pero no es obligatorio.
2. Usá la herramienta **AskUserQuestion** para pedir confirmación con botones —
   *"Voy a enviar este formulario con estos datos. ¿Le doy?"* con opciones **Sí / No**.
   Al usuario le aparece como un cuadro/modal donde toca una opción.
3. **No avances hasta que elija.** Si dice que no, pará y preguntá qué cambiar.

Las acciones reversibles (navegar, leer, escribir en un campo sin enviar, reproducir un video) no
necesitan confirmación — fluí normal y confirmá solo en el paso final irreversible.

## Robustez: manejá lo que se cruza

- **Diálogos de cookies / consentimiento.** Muchas webs muestran un "Antes de continuar…" o un banner
  de cookies que tapa todo. Si el `snapshot` muestra un botón de aceptar/rechazar/cerrar al inicio,
  resolvelo (normalmente aceptar o cerrar) ANTES de seguir, y avisá que lo hiciste.
- **Autoplay muteado.** Los navegadores a veces reproducen en silencio o en pausa. En tareas de
  "reproducir", si la verificación (regla 3) muestra `paused:true` o `muted:true`, dale play o
  desmuteá (click en el botón de play/sonido, o por teclado) y volvé a verificar.
- **Refs viejos.** Los `@e1, @e2, …` se renumeran en cada snapshot y quedan viejos apenas la página
  cambia. Volvé a hacer `snapshot -i` antes de la próxima acción con refs.

## El ciclo de trabajo

```
1. agent-browser --headed open <url>   → abre la VENTANA real (solo la primera vez)
2. agent-browser snapshot -i           → leé la página (TEXTO); resolvé cookies si las hay
3. (narrá la próxima acción en 1 línea)
4. agent-browser <acción> @eN ...       → ejecutá la acción (el usuario la ve en la ventana)
5. verificá el resultado si no es obvio (regla 3)
6. ¿es irreversible? → AskUserQuestion (Sí/No) ANTES de ejecutarla (regla 4)
7. volvé al paso 2 (re-snapshot: los @eN cambiaron)
```

## Comandos clave de agent-browser (chuleta)

```bash
agent-browser --headed open <url>       # abrir SIEMPRE con ventana real
agent-browser snapshot -i               # leer la página como texto (mi forma de navegar)
agent-browser click @e3                 # click
agent-browser fill @e2 "texto"          # borra y escribe
agent-browser type @e2 " más"            # escribe sin borrar
agent-browser press Enter               # tecla
agent-browser wait --load networkidle   # esperar que cargue
agent-browser wait --text "Listo"       # esperar a que aparezca un texto
agent-browser eval "<js>"               # verificar estado por texto (ej. video.paused)
agent-browser close                     # cerrar al terminar (no dejar colgado)
```

## Opcionales (solo a pedido, para no ensuciar el chat)

- **Video de la sesión** — un solo archivo para reverla: `agent-browser record start sesion.webm`
  … tarea … `agent-browser record stop`. Avisá dónde quedó el `.webm`.
- **Dashboard en vivo (localhost:4848)** — espejo en vivo + feed de comandos, en una pestaña del
  navegador: `agent-browser dashboard start`, abrir `http://localhost:4848`, apagar con
  `agent-browser dashboard stop`.
- **Captura puntual a un archivo** — `agent-browser screenshot --annotate paso.png` (guardada en disco,
  no volcada al chat) cuando necesites desambiguar o el usuario la pida.

## Al terminar

Cerrá el navegador (`agent-browser close`) para no dejar procesos colgando, y hacé un **resumen
breve en español**: qué se hizo, paso por paso. Si se grabó video, decí dónde quedó.

## Ejemplo: reproducir una canción en YouTube (sin imágenes en el chat)

**Tarea:** "Buscá 'y qué hay de mí' en YouTube y reproducila, quiero verlo paso a paso."

```
1. "Abro YouTube en una ventana para que la veas."
   agent-browser --headed open https://www.youtube.com

2. agent-browser snapshot -i        → veo el buscador @e12 (y reviso si hay banner de cookies)

3. "Escribo la canción en el buscador."
   agent-browser fill @e12 "y que hay de mi"
   "Apreto Enter para buscar."
   agent-browser press Enter
   agent-browser wait --load networkidle

4. agent-browser snapshot -i        → leo los resultados COMO TEXTO; identifico
   link "¿y que hay de mi?..." 4:23 [ref=e124]  ← el match por título

5. "Hago click en ese resultado para reproducirlo."
   agent-browser click @e124
   agent-browser wait --load networkidle

6. VERIFICO que suene (regla 3):
   agent-browser eval "...video.paused/muted..."   → paused:false, muted:false ✓
   (si estuviera muteado/pausado, le daría play/desmuteo y reverifico)

7. "Listo, está sonando 'Y qué hay de mí'." (cero capturas mandadas al chat)
```

El usuario vio TODO en la ventana real; el chat quedó limpio (solo narración en texto); y confirmé
que de verdad estaba sonando. Eso es lo que hace esta skill.
