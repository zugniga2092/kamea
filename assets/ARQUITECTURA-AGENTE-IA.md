# Arquitectura de Agente IA para Negocio Local
## Documento técnico — basado en implementación real para Kamea Gastro Bar

---

## 1. Qué es este sistema

Un agente de IA conversacional que atiende clientes 24/7 por Telegram (y en el futuro WhatsApp), gestiona reservas o citas, responde preguntas sobre el negocio, aprende de cada conversación y actúa como copiloto del dueño en modo admin.

**No es un bot de árbol de decisiones.** Es un agente que razona, mantiene contexto real entre mensajes, aprende preguntas nuevas y mejora con el uso.

---

## 2. Stack tecnológico

| Capa | Tecnología | Por qué |
|---|---|---|
| Runtime | Node.js ≥18 | Ligero, async nativo, perfecto para bots de mensajería |
| Mensajería | Telegraf (Telegram) → Twilio/360dialog (WhatsApp) | Fácil de migrar cambiando solo messenger.js |
| IA | Anthropic Claude API | Sonnet para conversaciones, Haiku para tareas rápidas (admin) |
| Base de datos | Supabase (PostgreSQL) | Persistencia de memoria, reservas, clientes, aprendizaje |
| Automatizaciones | n8n vía REST | Recordatorios, reviews, reactivación de clientes |
| Hosting | Railway.app | Deploy automático desde GitHub, variables de entorno fáciles |
| Versionado | GitHub (repo privado) | Un repo por cliente |

---

## 3. Estructura de archivos

```
agente/
├── index.js                  ← Entrada: arranca Express + bot
├── agent.js                  ← Núcleo: llama a Claude con memoria y contexto
├── messenger.js              ← Capa de mensajería (Telegram ahora, WhatsApp después)
├── memory.js                 ← Historial conversacional en Supabase
├── commands/
│   ├── cliente.js            ← Modo cliente: parsea etiquetas, gestiona reservas/citas
│   └── admin.js              ← Modo admin: comandos del dueño en lenguaje natural
└── workflows/
    └── n8n-endpoints.js      ← Endpoints REST para automatizaciones
```

**Regla de dependencias:**
- `index.js` importa `messenger.js` y `n8n-endpoints.js`
- `messenger.js` importa `cliente.js` y `admin.js`
- `cliente.js` importa `agent.js`
- `agent.js` importa `memory.js`
- **Nunca al revés** — evita dependencias circulares

---

## 4. Base de datos Supabase

Todas las tablas incluyen `business_id` para arquitectura multi-cliente (el mismo código sirve para diferentes negocios cambiando solo las variables de entorno).

### conversaciones
Memoria persistente del agente. Se recuperan los últimos 10 mensajes antes de cada respuesta.

```sql
id          uuid primary key default gen_random_uuid()
business_id text not null
chat_id     text not null
role        text not null  -- 'user' o 'assistant'
content     text not null
created_at  timestamptz default now()
```

### reservas / citas
Para una clínica: renombrar `reservas` a `citas` y adaptar campos.

```sql
id          uuid primary key default gen_random_uuid()
business_id text not null
chat_id     text not null
nombre      text not null
telefono    text
fecha_visita date not null
hora        text not null
personas    integer         -- para restaurante; para clínica: número de personas o eliminar
servicio    text not null   -- 'almuerzo'/'comida'/'cena' o 'consulta'/'tratamiento'/'revisión'
estado      text default 'pendiente'  -- 'pendiente','confirmada','rechazada','completada','no_show'
notas       text            -- alergias, preferencias, motivo de la visita
created_at  timestamptz default now()
```

### clientes
Memoria a largo plazo. El agente la consulta y ajusta su tono según el historial del cliente.

```sql
id            uuid primary key default gen_random_uuid()
business_id   text not null
chat_id       text not null unique
nombre        text
telefono      text
primera_visita date
ultima_visita  date
visitas_total  integer default 0
notas         text    -- preferencias, alergias, info relevante aprendida en conversaciones
activo        boolean default true
created_at    timestamptz default now()
```

### preguntas_desconocidas
Sistema de aprendizaje activo. El agente guarda lo que no sabe, avisa al dueño, y cuando el dueño responde queda en la base de conocimiento para siempre.

```sql
id          uuid primary key default gen_random_uuid()
business_id text not null
chat_id     text not null
pregunta    text not null
respuesta   text            -- se rellena cuando el dueño responde
estado      text default 'pendiente'  -- 'pendiente', 'respondida'
created_at  timestamptz default now()
```

### menu_dia / servicios_dia (adaptable)
Para restaurante: menú del día. Para clínica: disponibilidad del día, profesional de turno, promoción activa.

```sql
id              uuid primary key default gen_random_uuid()
business_id     text not null
fecha           date not null
-- Para restaurante:
entrante        text
plato_principal text
postre          text
precio          numeric default 12
agotados        text    -- platos no disponibles hoy
notas           text    -- avisos del día
terraza_abierta boolean
-- Para clínica: sustituir por campos como:
-- profesional    text
-- plazas_libres  integer
-- promocion      text
activo          boolean default true
created_at      timestamptz default now()
```

### notificaciones
Log de mensajes automáticos enviados por n8n.

```sql
id          uuid primary key default gen_random_uuid()
business_id text not null
tipo        text not null   -- 'recordatorio_24h', 'recordatorio_2h', 'resena', 'reactivacion'
contenido   text
destinatario text
enviado     boolean default false
created_at  timestamptz default now()
```

---

## 5. Cómo funciona el agente — flujo completo

```
Cliente escribe mensaje
        ↓
messenger.js recibe el mensaje
        ↓
¿Contiene #admin?
  ├── SÍ → admin.js (comandos del dueño)
  └── NO → cliente.js (conversación normal)
               ↓
        agent.js — construye system prompt:
          · Fecha de hoy (inyectada en tiempo real)
          · Información del negocio
          · Menú/servicios del día (consulta Supabase)
          · Perfil del cliente (consulta Supabase)
          · Reservas/citas activas del cliente
          · Base de conocimiento aprendida
               ↓
        Claude API (Sonnet) — genera respuesta
               ↓
        cliente.js — parsea etiquetas especiales:
          · [RESERVA: ...] → guarda en Supabase + notifica admin
          · [PREGUNTA_DESCONOCIDA: ...] → guarda + notifica admin
          · [CLIENTE_NOTA: ...] → actualiza perfil en Supabase
          · [CANCELAR_RESERVA: ...] → cancela en Supabase
          · [MODIFICAR_RESERVA: ...] → edita en Supabase
               ↓
        Respuesta al cliente (la etiqueta nunca llega al cliente)
```

---

## 6. El patrón de etiquetas — decisión arquitectónica clave

El agente no tiene herramientas ni function calling. En su lugar, Claude incluye etiquetas especiales en su respuesta que el código intercepta y procesa antes de enviar el texto al cliente.

**Ventajas:**
- Claude decide cuándo actuar (no hay lógica de if/else en el código para cada caso)
- Fácil de extender: añadir una etiqueta nueva = añadir una línea al system prompt y una al parser
- El cliente nunca ve la etiqueta — la experiencia es 100% conversacional

**Etiquetas del sistema Kamea:**

```
[RESERVA: nombre=X, fecha=DD/MM/YYYY, hora=HH:MM, personas=X, servicio=X, telefono=X, notas=X]
→ Guarda la reserva en Supabase con estado 'pendiente' y notifica al admin

[PREGUNTA_DESCONOCIDA: texto de la pregunta]
→ Guarda en preguntas_desconocidas y notifica al admin para que responda

[CLIENTE_NOTA: texto breve de lo aprendido]
→ Actualiza el campo notas del cliente en Supabase para futuras conversaciones

[CANCELAR_RESERVA: id=XXXXXXXX]
→ Cambia el estado de la reserva a 'cancelada' en Supabase

[MODIFICAR_RESERVA: id=XXXXXXXX, campo=hora, valor=15:30]
→ Edita el campo indicado de la reserva en Supabase
```

**Para una clínica estética, ejemplos de etiquetas equivalentes:**
```
[CITA: nombre=X, fecha=DD/MM/YYYY, hora=HH:MM, servicio=X, telefono=X, notas=X]
[PREGUNTA_DESCONOCIDA: texto]
[CLIENTE_NOTA: texto]
[CANCELAR_CITA: id=XXXXXXXX]
```

---

## 7. El system prompt — motor del comportamiento

El system prompt se construye en tiempo real en cada llamada a Claude (`buildSystemPrompt()` en `agent.js`). No es estático — se enriquece con datos de Supabase antes de enviarlo.

**Estructura del system prompt:**

```
1. FECHA DE HOY (inyectada en tiempo real — sin esto Claude usa su fecha de entrenamiento)
2. IDIOMA — detectar y mantener el idioma del cliente
3. QUIÉN ERES — persona real, no bot, tono del negocio
4. CÓMO HABLAS — estilo, longitud de respuestas, sin emojis o con ellos
5. LÍMITES — solo habla del negocio, cómo manejar preguntas ajenas
6. INFORMACIÓN DEL NEGOCIO — dirección, teléfono, horarios, servicios
7. CARTA / CATÁLOGO DE SERVICIOS — lo que ofrece el negocio con precios y detalles
8. CONTEXTO DEL DÍA — menú de hoy, disponibilidad, promoción activa (desde Supabase)
9. PERFIL DEL CLIENTE — nombre, visitas, notas (desde Supabase)
10. RESERVAS/CITAS ACTIVAS DEL CLIENTE — para que sepa sin preguntar (desde Supabase)
11. BASE DE CONOCIMIENTO — preguntas y respuestas aprendidas (desde Supabase)
12. VALIDACIÓN DE HORARIOS — qué días/horas acepta reservas, cuándo está cerrado
13. FLUJO DE RESERVA — cómo recoger datos, en qué orden, cómo no parecer un formulario
14. ETIQUETAS — cuándo y cómo escribir cada etiqueta
15. URGENCIAS — cuándo dar el teléfono directo
```

**Para una clínica estética, adaptar:**
- Sección 6: dirección, horarios de la clínica, profesionales disponibles
- Sección 7: catálogo de tratamientos con precios, duración y descripción
- Sección 8: disponibilidad del día, profesional de turno, promoción activa
- Sección 12: horarios de atención, días cerrados, tiempos mínimos entre citas
- Sección 13: flujo de cita (nombre, tratamiento, fecha, hora, teléfono)

---

## 8. Memoria a dos niveles

### Nivel 1 — Memoria conversacional (corto plazo)
Últimos 10 mensajes del chat, guardados en la tabla `conversaciones`. Se recuperan antes de cada respuesta. Sin esto el agente no recuerda lo que dijiste hace 3 mensajes.

### Nivel 2 — Perfil del cliente (largo plazo)
Tabla `clientes`. Persiste entre conversaciones y entre sesiones. El agente lo consulta al inicio de cada interacción y ajusta su tono:
- Si es cliente habitual → trato de confianza, no preguntar datos que ya sabe
- Si es primera vez → más explicativo, más cálido
- Si tiene alergias/preferencias anotadas → las usa sin que el cliente las repita

El agente también aprende activamente: cuando descubre algo relevante del cliente escribe `[CLIENTE_NOTA: ...]` y el sistema actualiza su perfil automáticamente.

---

## 9. Modo admin — copiloto del dueño

El dueño activa el modo admin escribiendo `#admin`. La sesión persiste 30 minutos sin actividad.

**Arquitectura del routing admin (`admin.js`):**
1. Regex matching directo para comandos conocidos (reservas, menú, confirmar, etc.)
2. Si no coincide ningún patrón → **fallback inteligente con Claude Haiku**: se le pasa el contexto real del negocio (reservas hoy/mañana, menú, preguntas pendientes) y Haiku interpreta la intención en lenguaje libre

Esto significa que el dueño nunca recibe un error — siempre obtiene una respuesta útil.

**Comandos tipo para cualquier negocio:**

| Comando | Acción |
|---|---|
| `#admin citas hoy` | Lista citas del día |
| `#admin citas mañana` | Lista citas del día siguiente |
| `#admin confirmar [nombre]` | Confirma cita pendiente y notifica al cliente |
| `#admin rechazar [nombre] [motivo]` | Rechaza y notifica |
| `#admin cancelar [nombre]` | Cancela cita confirmada |
| `#admin no-show [nombre]` | Marca como no presentado |
| `#admin preguntas` | Lista preguntas sin responder |
| `#admin respuesta [id] [respuesta]` | Responde y añade a la base de conocimiento |
| `#admin resumen` | Resumen completo del día |

---

## 10. Automatizaciones n8n

El agente expone un endpoint REST:
```
POST /agent
Body: { businessId, action, data }
```

n8n llama a este endpoint según triggers temporales. El bot procesa la acción y envía mensajes a los clientes correspondientes.

**Automatizaciones tipo para cualquier negocio:**

| Acción | Cuándo | Qué hace |
|---|---|---|
| `recordatorio_24h` | Día anterior a la cita | Mensaje personalizado al cliente |
| `recordatorio_2h` | 2h antes de la cita | Anti no-show |
| `solicitar_resena` | 48h después de la cita | Pide reseña en Google Maps |
| `reactivar_clientes` | Semanal | Mensaje a clientes inactivos >30 días |
| `reporte_semanal` | Lunes 9:00 | Resumen de la semana al dueño |
| `aviso_agenda` | Diario a las 9:00 | Avisa al dueño si el día está sin configurar |

---

## 11. Variables de entorno

```env
ANTHROPIC_API_KEY=        # API key de Anthropic
TELEGRAM_BOT_TOKEN=       # Token del bot creado con @BotFather
TELEGRAM_ADMIN_CHAT_ID=   # Chat ID del dueño del negocio
SUPABASE_URL=             # URL del proyecto Supabase del cliente
SUPABASE_ANON_KEY=        # Anon key del proyecto Supabase del cliente
BUSINESS_ID=nombre        # Identificador único del negocio (ej: 'clinica-estetica')
PORT=3000
```

---

## 12. Deploy en Railway

1. Crear repositorio privado en GitHub con el código del agente
2. Conectar el repositorio a Railway (detecta Node.js automáticamente)
3. Configurar las variables de entorno en Railway dashboard
4. Cada push a `main` despliega automáticamente
5. El bot corre en modo **polling** — no necesita webhook ni URL pública
6. La URL del servicio sirve el endpoint REST `/agent` para n8n

---

## 13. Cómo adaptar para una clínica estética

### Cambios en la base de datos
- Renombrar `reservas` → `citas`
- Campo `servicio`: `'consulta' | 'tratamiento' | 'revisión' | 'seguimiento'`
- Añadir campo `profesional` si hay varios
- En `menu_dia` → tabla `agenda_dia` con `profesional`, `plazas_libres`, `promocion`
- En `clientes` → añadir campos como `tipo_piel`, `tratamientos_anteriores`, `alergias_cosmeticos`

### Cambios en el system prompt
- Identidad: "Soy la asistente de [Nombre Clínica], especialistas en [especialidad]"
- Catálogo de tratamientos con precios, duración y descripción breve
- Contraindicaciones relevantes (embarazo, medicamentos, piel sensible)
- Política de cancelación (generalmente más estricta — mínimo 24h)
- Validación de citas: no hay citas en días/horas sin disponibilidad
- Tono: más cálido, más empático, hablar de bienestar y confianza

### Etiquetas a adaptar
```
[CITA: nombre=X, fecha=DD/MM/YYYY, hora=HH:MM, servicio=X, profesional=X, telefono=X, notas=X]
```

### Automatizaciones específicas para clínica
- Recordatorio 48h antes (más margen que en hostelería)
- Instrucciones previas al tratamiento (no depilarse X días antes, etc.)
- Seguimiento post-tratamiento (¿cómo fue la experiencia?)
- Reactivación con oferta personalizada según último tratamiento

### Lo que NO cambia
- `agent.js` — solo cambia el system prompt y las etiquetas
- `memory.js` — idéntico
- `messenger.js` — idéntico
- `admin.js` — solo cambia el vocabulario de los comandos
- El patrón de etiquetas — idéntico en concepto

---

## 14. Coste estimado mensual por cliente

| Servicio | Coste aprox. |
|---|---|
| Railway (hosting bot) | 5 $/mes |
| Supabase (BD) | Gratis hasta 500MB |
| Anthropic Claude API | 5-15 $/mes según volumen |
| n8n (self-hosted o cloud) | 0-20 $/mes |
| Telegram | Gratis |
| WhatsApp (360dialog) | 10-30 $/mes según mensajes |
| **Total estimado** | **20-70 $/mes** |

---

## 15. Checklist para arrancar un nuevo cliente

- [ ] Crear proyecto Supabase y ejecutar el SQL de las tablas
- [ ] Crear bot en Telegram con @BotFather
- [ ] Clonar el repositorio base con nuevo nombre
- [ ] Actualizar `BUSINESS_ID` y el system prompt en `agent.js` con info del negocio
- [ ] Crear repositorio privado en GitHub
- [ ] Crear servicio en Railway y configurar variables de entorno
- [ ] Verificar que el bot responde en Telegram
- [ ] Configurar workflows en n8n apuntando al endpoint del nuevo servicio
- [ ] Prueba completa con el dueño en modo admin
- [ ] Entrega y formación al dueño (30 minutos)


Lo que necesita una clínica estética (resumen ejecutivo)
Comunicación y trato

Agente que trate de usted / "estimado señor/a" — nunca tutear
Multiidioma: español e inglés mínimo
Tono cálido pero profesional

Gestión de citas

Recordatorios automáticos 24/48h antes
Reagendado y cancelación sin intervención humana
Lista de espera cuando hay cancelaciones

Seguimiento de tratamientos

Recordatorios según frecuencia del tratamiento: mensual, trimestral, anual
Post-tratamiento: instrucciones de cuidados específicas según lo que se hizo (postratamiento de láser ≠ postratamiento de bótox)
Encuesta de satisfacción automática tras cada visita

Captación y fidelización

Respuesta 24/7 a dudas sobre tratamientos, precios, contraindicaciones básicas
Cualificación del lead antes de proponer cita
Reactivación de pacientes inactivos
Sugerencia de tratamientos complementarios en el momento oportuno

Control y datos

Dashboard con leads por canal, conversiones, no-shows, ingresos por tratamiento
Informes automáticos semanales/mensuales para dirección