# 🪟 navegador-visible

Una skill para **Claude Code** que hace la automatización del navegador **transparente**: en vez de trabajar oculto, abre una **ventana de Chrome real** que mirás en vivo, **narra** cada paso en español, **verifica** que cada acción haya funcionado, y te pide **confirmación con botones** antes de cualquier acción irreversible.

Envuelve a [`agent-browser`](https://github.com/vercel-labs/agent-browser) (el CLI de automatización de Vercel) y le agrega una capa de **observabilidad y control** pensada para que veas y entiendas todo lo que hace el navegador.

> Ideal para quien no quiere que la IA "haga cosas por detrás" en sus cuentas, y para aprender mirando el paso a paso.

## ✨ Qué hace

| Regla | Comportamiento |
|---|---|
| 🪟 **Ventana real** | Abre Chrome con `--headed` en tu escritorio. Vos mirás esa ventana en vivo (no headless, no capturas en el chat). |
| 🗣️ **Narra paso a paso** | Antes de cada acción importante, dice en una línea qué va a hacer. |
| ✅ **Verifica** | Después de acciones cuyo éxito no es obvio (reproducir, loguear, guardar), comprueba que de verdad funcionó (`eval`/`wait`), no lo asume. |
| 🛑 **Confirma lo irreversible** | Antes de enviar formularios, comprar, borrar o mandar mensajes, pide confirmación con botones **Sí / No** y no avanza hasta que elijas. |
| 🔑 **Nunca escribe contraseñas** | Para entrar como un usuario de prueba carga una sesión ya iniciada (`state load`), y para cambiar de rol carga otra. El formulario de login lo probás vos. |

Por defecto **no manda imágenes al chat** (vos ves la ventana; la IA navega leyendo la página como texto). Opcionalmente puede grabar un **video** de la sesión o usar el **dashboard** de agent-browser.

## 📋 Requisitos

Esta skill **necesita el CLI `agent-browser` instalado** (es lo que maneja el navegador):

```bash
npm i -g agent-browser
```

- Node.js y npm.
- Google Chrome instalado (agent-browser usa tu Chrome del sistema).
- Probada con agent-browser **0.38.1**. Si tenés una versión vieja: `npm i -g agent-browser@latest`.

### ⚠️ Windows: dos bugs de agent-browser que la skill esquiva

| Bug | Qué hace la skill |
|---|---|
| Si la salida del primer comando pasa por un tubo (`\| tail`, `$(…)`), el comando no vuelve hasta que se cierra la ventana ([vercel-labs/agent-browser#1407](https://github.com/vercel-labs/agent-browser/issues/1407)) | Nunca usa tubos con agent-browser: salida directa o a un archivo |
| Con la ventana maximizada, `set viewport` responde `EOF … daemon may be busy` aunque el tamaño se aplica ([#1981](https://github.com/vercel-labs/agent-browser/issues/1981)) | Comprueba el tamaño con `eval` en vez de reintentar |

## 📥 Instalación

En Claude Code (PowerShell / terminal):

```bash
claude plugin marketplace add martin15006/navegador-visible
claude plugin install navegador-visible@navegador-visible
```

Reiniciá Claude Code para que la skill quede disponible.

## 🚀 Uso

No hace falta nombrar la skill: se activa sola cuando pedís algo de navegador y querés verlo. Por ejemplo:

> *"Buscá 'tal canción' en YouTube y reproducila, quiero verlo paso a paso."*

La skill abrirá una ventana real, irá narrando cada paso (abrir, buscar, escribir, click), verificará que el video esté sonando, y solo te frenará con un botón si hubiera algo irreversible.

## 🧩 ¿Cómo funciona por dentro?

Es una skill de **Claude Code** (un `SKILL.md` con instrucciones de comportamiento). No es código que se ejecuta: es un "manual" que Claude carga y sigue, usando los comandos de `agent-browser` (`--headed open`, `snapshot -i`, `click`, `fill`, `screenshot`, `record`, etc.).

```
navegador-visible/
├── .claude-plugin/
│   ├── marketplace.json   # para "claude plugin marketplace add"
│   └── plugin.json        # manifiesto del plugin
└── skills/
    └── navegador-visible/
        └── SKILL.md        # la skill en sí
```

## 📄 Licencia

[MIT](LICENSE) © 2026 Juan Sebastián Martín Moncada

## 🙌 Créditos

Construida sobre [`agent-browser`](https://github.com/vercel-labs/agent-browser) (Vercel). Hecha con [Claude Code](https://claude.com/claude-code) y su `skill-creator`.
