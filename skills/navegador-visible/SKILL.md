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
  botones antes de lo irreversible.
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
| `browser_take_screenshot` | `agent-browser screenshot archivo.png` (a disco, no al chat) |
| `browser_resize`, `resize_window` | `agent-browser set viewport <ancho> <alto>` (ej. 390 844 para móvil) |

- No uses un `--session` distinto al de la ventana: otra sesión es otro navegador, invisible.
- Los scripts de node o curl contra la API **sí** sirven para lo que la ventana no puede mostrar
  (permisos, casos límite, muchas validaciones seguidas). Van **después** de mostrar el flujo en la
  ventana, y avisando: *"Esto lo chequeo por API, no lo vas a ver en la ventana."*

## Regla 3: que la ventana muestre las acciones, no solo lecturas

`snapshot`, `eval` y `get` leen la página sin mover nada en pantalla. Sirven para decidir y
comprobar, pero lo que se está probando tiene que pasar de verdad en la ventana:

- Si probás un flujo (login → dashboard → crear algo), hacelo con clicks y campos en la ventana, no
  saltando con `open` a la URL del final.
- **Después de editar código**, `agent-browser reload` (o `open` de la misma URL) y comprobá que el
  cambio está. Sin eso la ventana sigue mostrando la versión vieja y parece congelada.
- Si reiniciaste el servidor en otro puerto, `open` de la URL nueva **en la misma ventana**.

## Regla 4: narrá una línea antes de actuar

Antes de cada paso importante, una línea en español: qué vas a hacer y por qué. *"Escribo el
usuario de prueba y entro."* No describas cada tecla: el usuario ya ve la ventana. Si pide más o
menos detalle, ajustá.

## Regla 5: comprobá que funcionó

Si el éxito no se ve obvio, comprobalo en vez de suponerlo:

- Login o navegación: `agent-browser wait --url "**/dashboard"` o `wait --text "Bienvenido"`.
- Guardar o enviar: `agent-browser wait --text "Guardado"`.
- Video o audio sonando:
  ```bash
  agent-browser eval "(() => { const v = document.querySelector('video'); return v ? JSON.stringify({paused:v.paused, muted:v.muted, t:Math.round(v.currentTime)}) : 'no-video'; })()"
  ```

Si la comprobación falla, decilo y corregí. No anuncies éxito sin prueba.

## Regla 6: confirmá con botones antes de lo irreversible

Enviar un formulario real, comprar, borrar, mandar un mensaje, pagar, aceptar términos:

1. Decí en palabras qué está por pasar (qué datos, qué botón). El usuario lo ve en la ventana.
2. Preguntá con **AskUserQuestion**: *"Voy a enviar este formulario con estos datos. ¿Le doy?"*,
   opciones **Sí / No**.
3. No avances hasta que elija. Si dice que no, preguntá qué cambiar.

Navegar, leer, escribir en un campo sin enviar o reproducir un video no necesitan confirmación.

## Robustez

- **Cookies o "antes de continuar".** Si el `snapshot` muestra un aviso que tapa la página,
  resolvelo primero y avisá que lo hiciste.
- **Autoplay muteado.** Si la comprobación da `paused:true` o `muted:true`, dale play o desmuteá y
  volvé a comprobar.
- **Refs viejos.** Los `@eN` se renumeran cuando la página cambia. `snapshot -i` de nuevo antes de
  usar refs.
- **Logins con cuenta real (Google, bancos, etc.).** No escribas contraseñas en esta ventana: la
  controla la automatización y Google suele bloquear ese login. Pedile al usuario que inicie sesión
  él en la ventana, o que use su navegador normal.

## Capturas

Por defecto, ninguna al chat: el usuario ya ve la ventana y las imágenes solo alargan la
conversación. Una sola, guardada a disco, si hace falta desambiguar algo o si el usuario la pide.
Para un registro completo está el video (ver Opcionales).

## Ciclo de trabajo

```
1. agent-browser --headed open <url>     → aparece la ventana (avisale)
2. agent-browser snapshot -i             → leés la página como texto; cookies si hay
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
agent-browser wait --text "Listo"     # esperar un texto
agent-browser eval "<js>"             # comprobar estado
agent-browser screenshot paso.png     # captura a disco (no al chat)
agent-browser close                   # cerrar al final
```

## Opcionales (a pedido)

- **Video de la sesión:** `agent-browser record start sesion.webm` … `agent-browser record stop`.
  Decí dónde quedó el archivo.
- **Dashboard en vivo:** `agent-browser dashboard start`, abrir `http://localhost:4848`, y al final
  `agent-browser dashboard stop`.

## Al terminar

Cerrá la ventana (`agent-browser close`), apagá el servidor que levantaste y resumí en español qué
se probó y qué dio.

## Ejemplo: probar un login en una app local

```
(servidor corriendo en :3000)
1. "Abro tu app en una ventana para que la veas."
   agent-browser --headed open http://localhost:3000/login
2. agent-browser snapshot -i              → email @e3, contraseña @e4, botón @e5
3. "Escribo el usuario de prueba y entro."
   agent-browser fill @e3 "admin@demo.com"
   agent-browser fill @e4 "Demo123*"
   agent-browser click @e5
   agent-browser wait --url "**/dashboard"      ✓ entró
4. Aparece un error en el dashboard; edito el código…
   "Recargo para que veas el arreglo."
   agent-browser reload
   agent-browser wait --text "Proyectos"        ✓ ya se ve
5. "Ahora chequeo por API que un usuario sin permiso no pueda borrar. Esto no se ve en la ventana."
   node scripts/probar-permisos.mjs             (complemento, después del flujo visible)
6. agent-browser close, apago el servidor, resumen.
```

En ningún momento se usó Playwright ni el panel integrado: todo lo visible pasó en la ventana.
