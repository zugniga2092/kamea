# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

# Repositorio — Estructura General

Este repositorio contiene dos proyectos independientes más una web estática:

| Directorio / archivos | Proyecto | Estado en git |
|---|---|---|
| `*.html` (raíz) | Web de Kamea Gastro Bar | Tracked |
| `bot/` | Bot Telegram de **Bodega Ruzafa** | Tracked |
| `kamea-bot/` | Bot Telegram/WhatsApp de **Kamea Gastro Bar** | No tracked (desarrollo local) |

Los dos bots comparten el mismo patrón de etiquetas y stack tecnológico, pero tienen arquitecturas distintas.

---

## Web Kamea (raíz)

Archivos HTML estáticos: `index.html`, `aviso-legal.html`, `politica-cookies.html`, `politica-privacidad.html`. Sin build step ni dependencias. Edición directa.

Recursos estáticos en `assets/`: imágenes de galería y hero, logo, favicon, y PDFs de las cartas (`carta-almuerzos.pdf`, `carta-cocina.pdf`).

---

## Bot Bodega Ruzafa (`bot/`)

Bot de Telegram para una bodega boutique en Ruzafa, Valencia. Arquitectura monolítica — toda la lógica en `bot/index.js`.

### Comandos

```bash
cd bot
npm install        # requiere Node >=18
npm run dev    # node --watch index.js
npm start      # node index.js
```

El bot arranca siempre en **modo polling** (no webhook). No necesita URL pública para desarrollo local ni en Render.

### Variables de entorno (`bot/.env`)

```env
TELEGRAM_TOKEN=       # Token de @BotFather
ANTHROPIC_API_KEY=    # Cerebro del agente
ADMIN_CHAT_ID=        # Chat ID de Jairo (dueño) para comandos admin y notificaciones
SUPABASE_URL=
SUPABASE_KEY=         # Anon key de Supabase
PORT=3000
```

### Arquitectura

- **Memoria conversacional:** `Map<chatId, messages[]>` en RAM. Máximo 20 mensajes. Se pierde al reiniciar — diseño intencional para esta versión.
- **Supabase:** Solo persiste clientes y reservas (no el historial de conversación).
- **Agente IA:** `claude-sonnet-4-6` con system prompt generado en cada llamada (`buildSystemPrompt()` consulta la carta de vinos en Supabase en tiempo real).
- **Endpoints REST** (llamados desde cron-job.org, no n8n):
  - `GET /recordatorios` — Recordatorios 24h antes de la reserva (cron: cada hora)
  - `GET /reviews` — Solicitud de valoración 48h después (cron: cada hora)
  - `GET /reactivar` — Reactivación de clientes inactivos 30+ días (cron: semanal)
- **Deploy:** Render.com (Root Directory: `bot`, Build: `npm install`, Start: `node index.js`)

### Comandos admin (solo para ADMIN_CHAT_ID)

Son palabras clave en mayúsculas, sin pasar por Claude:

| Mensaje | Acción |
|---|---|
| `JAIRO2024` | Lista reservas de hoy |
| `SEMANA` | Reservas confirmadas de la semana |
| `CLIENTES` | Total de clientes en BD |
| `CANCELAR N` | Cancela la reserva número N de hoy y notifica al cliente |
| `PROMO` | Notifica al admin para enviar una oferta (**sin restricción de admin** en el código — cualquier usuario puede activarlo) |
| `AYUDA` | Lista de comandos disponibles |

### Patrón de etiqueta de reserva

Cuando Claude confirma una reserva emite esta etiqueta (y nada más):

```
[RESERVA: nombre=X, fecha=DD/MM/YYYY, hora=HH:MM, personas=X, telefono=X]
```

`index.js` intercepta la etiqueta, guarda en Supabase con `estado='confirmada'`, notifica al admin y responde al cliente con "Perfecto, todo anotado. Nos vemos pronto." La etiqueta nunca llega al cliente. Después inyecta un mensaje de sistema en el historial para que Claude no repita la etiqueta.

---

## Bot Kamea Gastro Bar (`kamea-bot/`)

Bot más complejo para el restaurante Kamea. No está en git — se desarrolla localmente y tiene su propio repositorio privado (`github.com/zugniga2092/kamea-bot`).

### Comandos

```bash
cd kamea-bot
npm install
npm run dev    # node --watch index.js
npm start
```

### Variables de entorno (`kamea-bot/.env`)

```env
ANTHROPIC_API_KEY=
TELEGRAM_BOT_TOKEN=      # Nombre distinto al del bot de Bodega Ruzafa
TELEGRAM_ADMIN_CHAT_ID=  # Chat ID de Alex (dueño)
SUPABASE_URL=
SUPABASE_ANON_KEY=       # Nombre distinto al del bot de Bodega Ruzafa
BUSINESS_ID=kamea
PORT=3000
```

### Diferencias arquitectónicas respecto a `bot/`

| Aspecto | `bot/` (Bodega Ruzafa) | `kamea-bot/` (Kamea) |
|---|---|---|
| Estructura | Monolítica (1 archivo) | Modular (`agent.js`, `memory.js`, `messenger.js`, `commands/`) |
| Memoria conversacional | RAM (Map, se pierde al reiniciar) | Supabase (persistente entre reinicios) |
| Admin | Palabras clave en mayúsculas | `#admin` en lenguaje natural + Haiku como fallback |
| Sesiones admin | No existe modo admin explícito | Map en RAM, expira tras 30 min de inactividad |
| Automatizaciones | cron-job.org via GET endpoints | n8n via `POST /agent` con `{ businessId, action, data }` |
| Deploy | Render.com | Railway.app |
| Tablas extra en Supabase | — | `preguntas_desconocidas`, `menu_dia`, `notificaciones` |

### Arquitectura modular de `kamea-bot/`

```
index.js          → Entrada: inicializa Express + Telegraf
agent.js          → Llamada a Claude API; inyecta fecha actual en el system prompt
messenger.js      → Capa Telegraf (futura migración a WhatsApp en este archivo)
memory.js         → getHistory / saveMessage / clearHistory en Supabase
commands/
  cliente.js      → Parsea etiquetas [RESERVA:] y [PREGUNTA_DESCONOCIDA:] del output de Claude
  admin.js        → Regex matching para comandos #admin; fallback a Haiku para lenguaje libre
workflows/
  n8n-endpoints.js → Endpoints REST para automatizaciones n8n
```

**Importante — dependencia circular:** `sendSafe()` está definida inline en `cliente.js` y no se importa desde `messenger.js`, porque `messenger.js` importa `cliente.js`. Importarla en sentido inverso causaría ciclo.

**Importante — formato de fecha en reservas:** El system prompt instruye a Claude a escribir `DD/MM/YYYY` en la etiqueta `[RESERVA:]`, pero Supabase espera `YYYY-MM-DD`. `cliente.js:guardarReserva` pasa la fecha sin convertir — si las fechas llegan mal a Supabase, aquí está el motivo.

**Validación de horarios:** No hay validación en código. Se hace vía instrucciones en el system prompt (domingos cerrado, cenas solo viernes/sábado, rangos de hora).

### Etiquetas que parsea `cliente.js`

```
[RESERVA: nombre=X, fecha=DD/MM/YYYY, hora=HH:MM, personas=X, servicio=X, telefono=X, notas=X]
[PREGUNTA_DESCONOCIDA: texto de la pregunta]
```

Las etiquetas son interceptadas antes de llegar al cliente. El contrato formato-etiqueta existe entre el system prompt (`agent.js`) y el parser (`cliente.js`) — cambiar uno exige cambiar el otro.

---

## Base de Datos Supabase

Cada bot usa su propia instancia de Supabase. Todas las tablas tienen `business_id` para arquitectura multi-cliente.

`bot/` (Bodega Ruzafa) usa: `clientes` (campos relevantes: `chat_id`, `nombre`, `telefono`, `ultima_visita`, `ultima_reactivacion`), `reservas` (campos relevantes: `recordatorio_enviado`, `review_enviado`, `estado`), `vinos` (campos: `nombre`, `variedad`, `precio`, `descripcion`, `activo`).

`kamea-bot/` (Kamea) usa: `conversaciones`, `clientes`, `reservas`, `preguntas_desconocidas`, `menu_dia`, `notificaciones`.

---

## Cómo clonar para un nuevo cliente

Solo cambian las variables de entorno y el system prompt en `agent.js`. El código no cambia. Nuevo BUSINESS_ID → nueva instancia Supabase → nuevo token Telegram → nuevo repositorio → nuevo servicio en Railway/Render.

---

