---
name: navegador-visible
description: >-
  Probar o usar CUALQUIER cosa con pantalla en una ventana REAL de Chrome que el usuario ve en su
  escritorio (agent-browser --headed), narrando cada paso. Úsala SIEMPRE que haya un navegador de
  por medio: cuando levantes un servidor o una app web para probarla (login, formularios, que la
  página cargue, QA, verificar un cambio de UI), aunque el usuario no lo pida en ese mensaje; y
  cuando diga "mostrame", "que yo lo vea", "en vivo", "abrí la página", "probalo", "buscá en tal
  web". Reemplaza al panel Browser integrado (mcp__Claude_Browser), a Claude Preview y al MCP de
  Playwright para todo lo que tenga pantalla, porque esos corren donde el usuario no mira. Una vez
  abierta la ventana, toda la tarea de UI sigue en ella hasta el final. Pide confirmación con
  botones antes de lo irreversible. También cuando agent-browser se cuelga, no vuelve o responde
  "daemon may be busy or unresponsive".
license: MIT
---

# Navegador visible

## Lo esencial (si solo leés esto, alcanza)

1. Primer comando: `agent-browser --headed open <url>`. Sin `--headed` la ventana no aparece.
2. **Toda** la prueba de UI de la tarea va por esa ventana. Nada de Playwright MCP, panel Browser,
   Claude Preview ni capturas por detrás.
3. ¿Editaste código? `agent-browser reload`, así la ventana muestra la versión nueva.
4. No cierres la ventana a mitad de tarea. Si se cerró, reabrí **con** `--headed`.
5. Scripts contra la API: solo de complemento, avisando *"esto no se ve en la ventana"*.
6. Una línea de narración antes de cada paso; botones de confirmación antes de lo irreversible.
7. **Nunca pases la salida de `agent-browser` por un tubo** (`| tail`, `| head`, `| grep`, `$(…)`,
   `$x = …` en PowerShell). Que salga directa o a un archivo, y filtrá el archivo.
8. ¿Se colgó o dice `daemon may be busy or unresponsive`? No reintentes `open`, no cambies de
   `--session` y no cierres la ventana: seguí [Si se cuelga o da error](#si-se-cuelga-o-da-error).
9. **No escribas contraseñas en la ventana, ni siquiera de usuarios de prueba.** Para entrar,
   cargá una sesión: [Entrar como un usuario de prueba](#entrar-como-un-usuario-de-prueba).

## Por qué existe

El usuario sigue lo que hacés **mirando la ventana de Chrome en su escritorio**. Si la prueba corre
en otro lado, para él la ventana queda congelada y parece que dejaste de trabajar. Ya pasó así,
medido en sesiones reales:

- Se hicieron cientos de pasos con Playwright o con el panel integrado (728 en una sesión, 49 en
  otra) antes de acordarse de esta skill. El usuario no vio nada.
- En otra sesión se abrió la ventana, se cerró, y la prueba siguió 188 pasos en Playwright mientras
  la ventana de Chrome quedaba quieta.

La ventana es la única pantalla que el usuario mira. Por eso todo lo visible pasa por ahí.

## Regla 1: ventana real desde el primer comando

```bash
agent-browser --headed open http://localhost:3000
```

- agent-browser abre el navegador **oculto** por defecto. `--headed` es lo que hace aparecer la ventana.
- Mientras la ventana siga abierta, los comandos siguientes la reutilizan aunque no lleven
  `--headed` (verificado).
- Si la cerraste, se cayó o reiniciaste agent-browser, el próximo `open` sin `--headed` arranca un
  navegador **invisible**. Siempre reabrí con `--headed`.
- Avisale que se abrió. Si no la ve, seguramente quedó detrás de la app de Claude (Alt+Tab).

## Regla 2: una sola ventana para toda la tarea

Una vez abierta, **cada paso de UI** va por agent-browser en esa ventana: navegar, hacer click,
llenar, comprobar lo que hay en pantalla.

No cambies a `mcp__plugin_playwright_playwright__*`, `mcp__Claude_Browser__*` ni
`mcp__Claude_Preview__*` a mitad de camino. Todo lo que hacen tiene equivalente acá:

| Si ibas a usar… | En esta skill |
|---|---|
| `browser_navigate`, `navigate` | `agent-browser open <url>` |
| `browser_click`, `click` | `agent-browser click @eN` |
| `browser_type`, `browser_fill_form`, `form_input` | `agent-browser fill @eN "texto"` |
| `browser_snapshot`, `read_page` | `agent-browser snapshot -i` |
| `browser_evaluate`, `javascript_tool` | `agent-browser eval "<js>"` |
| `browser_take_screenshot` | `agent-browser screenshot "$TEMP/paso.png"` (fuera del proyecto, no al chat) |
| Mirar un archivo suelto (un SVG, un HTML de prueba) | `agent-browser open "file:///C:/ruta/icono.svg"`, en la misma ventana |
| `browser_resize`, `resize_window` | `agent-browser set viewport <ancho> <alto>` (ej. 390 844 para móvil). Con la ventana maximizada da error aunque funcione: ver [Si se cuelga o da error](#si-se-cuelga-o-da-error) |

- No uses un `--session` distinto al de la ventana: otra sesión es otro navegador, invisible.
- Los scripts de node o curl contra la API **sí** sirven para lo que la ventana no puede mostrar
  (permisos, casos límite, muchas validaciones seguidas). Van **después** de mostrar el flujo en la
  ventana, y avisando: *"Esto lo chequeo por API, no lo vas a ver en la ventana."*

## Regla 3: que la ventana muestre las acciones, no solo lecturas

`snapshot`, `eval` y `get` leen la página sin mover nada en pantalla. Sirven para decidir y
comprobar, pero lo que se está probando tiene que pasar de verdad en la ventana:

- Si probás un flujo (dashboard → crear algo → verlo en la lista), hacelo con clicks y campos en la
  ventana, no saltando con `open` a la URL del final.
- **Después de editar código**, `agent-browser reload` (o `open` de la misma URL) y comprobá que el
  cambio está. Sin eso la ventana sigue mostrando la versión vieja y parece congelada.
- Si reiniciaste el servidor en otro puerto, `open` de la URL nueva **en la misma ventana**.

## Regla 4: narrá una línea antes de actuar

Antes de cada paso importante, una línea en español: qué vas a hacer y por qué. *"Entro como el
Coordinador de prueba para ver su menú."* No describas cada tecla: el usuario ya ve la ventana. Si pide más o
menos detalle, ajustá.

## Regla 5: comprobá que funcionó

Si el éxito no se ve obvio, comprobalo en vez de suponerlo:

- Navegación: `agent-browser wait --url "**/dashboard"` o `wait --text "Bienvenido"`.
- Guardar o enviar: `agent-browser wait --text "Guardado"`, con el texto que muestra la app de
  verdad (mirá el código si no lo sabés).
- **Esperá con `wait`, no con `sleep`:** `sleep` adivina el tiempo y `wait` espera lo que tiene que
  aparecer (`wait 2000` también es adivinar). Si el aviso se va solo a los pocos segundos, poné el
  click y la espera en el mismo comando:
  `agent-browser click @e12 && agent-browser wait --text "Guardado"`. `wait` corta a los 25 s: si
  corta, lo que esperabas no pasó, y eso es un resultado.
- Video o audio sonando:
  ```bash
  agent-browser eval "(() => { const v = document.querySelector('video'); return v ? JSON.stringify({paused:v.paused, muted:v.muted, t:Math.round(v.currentTime)}) : 'no-video'; })()"
  ```

Si la comprobación falla, decilo y corregí. No anuncies éxito sin prueba.

## Regla 6: confirmá con botones antes de lo irreversible

Enviar un formulario real, comprar, borrar, mandar un mensaje, pagar, aceptar términos:

1. Decí en palabras qué está por pasar (qué datos, qué botón). El usuario lo ve en la ventana.
2. Preguntá con **AskUserQuestion**: *"Voy a enviar este formulario con estos datos. ¿Le doy?"*,
   opciones **Sí / No**. Si no tenés esa herramienta, hacé la pregunta en texto y frená.
3. No avances hasta que elija. Si dice que no, preguntá qué cambiar.

Navegar, leer, escribir en un campo sin enviar o reproducir un video no necesitan confirmación.
Guardar datos de un usuario de prueba en una base local tampoco. En la base de producción, sí.

## Entrar como un usuario de prueba

**No escribas contraseñas en la ventana, ni siquiera de usuarios de prueba.** Es una regla de
seguridad fija: ninguna skill ni ningún permiso la cambia. Para entrar, cargá una sesión ya
iniciada:

1. **Conseguí la sesión**, de una de estas dos formas:
   - Un script (del proyecto, o tuyo en `$TEMP`) inicia sesión **por la API** con el usuario de
     prueba de ese rol y escribe el archivo de sesión. Mirá el código de login de la app para
     saber qué guarda y dónde (un token en `localStorage`, cookies…).
   - Martín entra una vez en la ventana con ese usuario y vos guardás la sesión:
     `agent-browser state save "$TEMP/sesion-coordinador.json"`.

   El archivo tiene este formato (el `origin` tiene que ser exactamente el de la app):
   ```json
   {"cookies": [], "origins": [{"origin": "http://localhost:5173",
     "localStorage": [{"name": "token", "value": "<token>"}]}]}
   ```
2. **Cargala en la ventana:** `agent-browser state load "$TEMP/sesion-coordinador.json"` y después
   `agent-browser open <una página de la app>`. La sesión se aplica al abrir la página, venga la
   ventana de donde venga.
3. **Para cambiar de rol:** cerrá la sesión con el botón de la app (así se borra lo que la app
   guardó del usuario anterior, como la empresa elegida), cargá el archivo del otro rol y abrí la
   página.

Cuidados:

- Los archivos de sesión y las contraseñas de prueba van en `$TEMP` o en un archivo que git
  ignore. Comprobalo en el `.gitignore`: si no lo ignora, avisale a Martín antes de que haga su
  próximo commit. Nunca en el repo ni en el chat, y no les hagas `cat`. Al terminar, borrá los
  archivos de sesión: tienen tokens.
- Los usuarios de prueba llevan un nombre que se note (`TST …`, `ZZ-PRUEBA …`). ¿La app usa la base
  de datos de producción? Mirá la URL de la base en el `.env` del backend o preguntale a Martín.
  Si la usa: nunca superadmin, y los usuarios de prueba se borran al terminar.
- **El formulario de login en sí** (escribir usuario y contraseña, el error si está mal) lo prueba
  Martín. En un pacto, ese criterio va como `[Martín]`.

## Robustez

- **Cookies o "antes de continuar".** Si el `snapshot` muestra un aviso que tapa la página,
  resolvelo primero y avisá que lo hiciste.
- **Autoplay muteado.** Si la comprobación da `paused:true` o `muted:true`, dale play o desmuteá y
  volvé a comprobar.
- **Refs viejos.** Los `@eN` se renumeran cuando la página cambia. `snapshot -i` de nuevo antes de
  usar refs.
- **`wait --text` busca el texto como se ve**, no como está en el HTML. Si el CSS lo pone en
  mayúsculas (`text-transform`), esperá `"ELIMINAR LA EMPRESA"`, no `"Eliminar la empresa"`.
- **Campos de fecha (`<input type="date">`).** `fill` y `keyboard type` no escriben nada, y `click`
  abre el calendario, que se queda con las teclas. Lo que funciona: `agent-browser focus "#fecha"` y
  después un `press` por dígito, en el orden que muestra el campo (en Colombia, día/mes/año:
  `press 2`, `press 3`, `press 0`, `press 9`…). Comprobalo con `eval` del `value`.
- **Git Bash cambia los argumentos que empiezan con `/`.** `/admin/usuarios` llega como
  `C:/Program Files/Git/admin/usuarios`, y lo mismo le pasa a un `eval` o un `wait --fn` que
  empieza con una regex `/…/`. Usá la URL completa y empezá los comandos con
  `export MSYS_NO_PATHCONV=1`.
- **Cuentas reales (Google, bancos, las de Martín).** Nada de sesiones guardadas ni scripts:
  pedile que inicie sesión él en la ventana, o que use su navegador normal. Google además suele
  bloquear los logins en ventanas automatizadas.

## Si se cuelga o da error

agent-browser tiene tres bugs que siguen sin arreglo (comprobados en la 0.38.1):

| Qué pasa | Por qué | Qué hacer |
|---|---|---|
| A mitad de la prueba la página queda en `about:blank` y se perdió la sesión. A veces la salida dice `[agent-browser] launched browser` | Antes de cada comando, agent-browser le da a Chrome **3 segundos** para contestar. Si la PC está cargada (poca RAM libre, un build o tests pesados corriendo) y tarda más, lo da por muerto y abre otro Chrome en blanco. **No es culpa del último comando**: le pasa a cualquiera, `wait` incluido | Volvé a abrir la página y a cargar la sesión, **una** vez, y avisale a Martín que la ventana se reinició. Si vuelve a pasar, bajá la carga: no corras builds, tests pesados ni instalaciones mientras usás la ventana, y apagá servidores y ventanas que no uses. |
| Un comando no vuelve (Bash lo pasa a segundo plano), pero la ventana sí se abrió | Su salida iba por un tubo y ese comando arrancó la sesión: agent-browser deja el tubo abierto hasta que la ventana se cierra | La ventana anda. Comprobalo con `agent-browser get url` (sin tubo) y seguí desde ahí, sin tubos. No reabras. Dejá el comando colgado en segundo plano: termina solo cuando cerrás la ventana al final. |
| `set viewport` responde `EOF while parsing … daemon may be busy or unresponsive` | La ventana está maximizada | Comprobá el tamaño con `agent-browser eval "innerWidth+'x'+innerHeight"`. Casi siempre se aplicó igual y seguís. Si no, pedile a Martín que la desmaximice, repetí `set viewport` **una** vez y comprobá de nuevo. Si sigue sin aplicarse, frená y contale. |

Cualquier comando puede arrancar la sesión: el primero, o el que llega después de que la ventana
se cerró o se cayó. Por eso la regla es **ningún tubo, nunca**, no "solo en el primero".
Para que un comando no espere de más, usá el timeout de la herramienta Bash.

**Para filtrar una salida larga**, mandala a un archivo fuera del proyecto y filtrá el archivo:

```bash
agent-browser snapshot -i > "$TEMP/snap.txt" 2>&1
grep -n "Crear vehículo" "$TEMP/snap.txt"

# y para guardar un valor en una variable, en vez de $(…):
agent-browser get text ".contador" > "$TEMP/valor.txt" 2>&1
read -r valor < "$TEMP/valor.txt"
```

`timeout 60 agent-browser … | tail` **no** protege: `timeout` corta agent-browser, pero `tail`
sigue esperando.

**Si ni `agent-browser get url` responde** (recién ahí se cierra la ventana):

1. No reintentes `open` en bucle ni pruebes otra `--session`: cada intento deja un proceso y una
   ventana más. Con muchos, Windows se queda sin recursos y hasta bash falla
   (`fork: Resource temporarily unavailable`).
2. `agent-browser session list` para ver qué quedó abierto.
3. Avisale a Martín que vas a cerrar la ventana, corré `agent-browser close --all` y reabrí
   **una sola vez** con `--headed`, en la última URL que viste. Antes de repetir la acción que
   falló, mirá si llegó a hacerse (un click puede haber pasado aunque dio error).
4. Si sigue fallando, frená y contale a Martín el error tal cual.

## Capturas

Por defecto, ninguna al chat: el usuario ya ve la ventana y las imágenes solo alargan la
conversación. Una sola, si hace falta desambiguar algo o si el usuario la pide, y siempre en
`$TEMP`: con una ruta relativa cae dentro del proyecto. Para un registro completo está el video
(ver Opcionales).

## Ciclo de trabajo

```
1. agent-browser --headed open <url>     → aparece la ventana (avisale)
2. agent-browser snapshot -i             → leés la página como texto (sin tubos); cookies si hay
3. narración de una línea
4. agent-browser click / fill / press    → la acción pasa en la ventana
5. comprobás el resultado (wait / eval)
6. ¿es irreversible? → AskUserQuestion antes
7. ¿editaste código? → agent-browser reload
8. volvés a 2 (los @eN cambiaron)
9. al final: agent-browser close + apagar el servidor + resumen
```

## Chuleta

```bash
agent-browser --headed open <url>     # abrir (siempre con --headed)
agent-browser snapshot -i             # leer la página como texto
agent-browser click @e3               # click
agent-browser fill @e2 "texto"        # borrar y escribir
agent-browser press Enter             # tecla
agent-browser reload                  # recargar después de editar código
agent-browser back                    # volver atrás
agent-browser wait --load networkidle # esperar a que cargue
agent-browser wait --text "Listo"     # esperar un texto (en vez de sleep)
agent-browser eval "<js>"             # comprobar estado
agent-browser state load "$TEMP/sesion-rol.json"  # entrar como un usuario de prueba
agent-browser screenshot "$TEMP/paso.png"         # captura fuera del proyecto (no al chat)
agent-browser session list            # qué sesiones hay abiertas
agent-browser close                   # cerrar al final (close --all si quedaron varias)
```

Ninguno va con `| tail`, `| head`, `| grep` ni `$(…)`: salida directa, o a un archivo y después
filtrás el archivo.

## Opcionales (a pedido)

- **Video de la sesión:** `agent-browser record start sesion.webm` … `agent-browser record stop`.
  Decí dónde quedó el archivo.
- **Dashboard en vivo:** `agent-browser dashboard start`, abrir `http://localhost:4848`, y al final
  `agent-browser dashboard stop`.

## Al terminar

Cerrá la ventana (`agent-browser close`), apagá el servidor que levantaste y resumí en español qué
se probó y qué dio.

## Ejemplo: probar una app local con usuario de prueba

```
(servidor corriendo en :3000; node scripts/sesion-prueba.mjs admin ya dejó $TEMP/sesion-admin.json)
1. "Abro tu app en una ventana para que la veas."
   agent-browser --headed open http://localhost:3000/login
2. "Entro como el administrador de prueba."
   agent-browser state load "$TEMP/sesion-admin.json"
   agent-browser open http://localhost:3000/dashboard
   agent-browser wait --text "Proyectos"        ✓ entró
3. Aparece un error en el dashboard; edito el código…
   "Recargo para que veas el arreglo."
   agent-browser reload
   agent-browser wait --text "Proyectos"        ✓ ya se ve
4. "Ahora chequeo por API que un usuario sin permiso no pueda borrar. Esto no se ve en la ventana."
   node scripts/probar-permisos.mjs             (complemento, después del flujo visible)
5. agent-browser close, apago el servidor, resumen.
```

En ningún momento se usó Playwright ni el panel integrado: todo lo visible pasó en la ventana.
