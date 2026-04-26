# Manual del asistente WhatsApp — Kamea Gastro Bar

El asistente atiende a los clientes 24h de forma automática. Tú lo controlas desde tu WhatsApp con comandos que empiezan por `#admin`.

---

## Rutina diaria recomendada

### Cada mañana (antes de abrir)
```
#admin menú hoy Entrante: gazpacho. Principal: merluza a la plancha. Postre: flan casero.
#admin terraza abierta
```
*(o `#admin terraza cerrada` si no está disponible)*

**Importante:** escribir algo al bot cada mañana mantiene activa la comunicación y garantiza que las notificaciones de reservas te lleguen durante el día.

### Durante el servicio
```
#admin agotado merluza
```

### Al cerrar
```
#admin resumen
```

---

## Reservas

| Lo que quieres hacer | Comando |
|---|---|
| Ver reservas de hoy | `#admin reservas hoy` |
| Ver reservas de mañana | `#admin reservas mañana` |
| Ver reservas de esta semana | `#admin reservas semana` |
| Ver reservas de un día concreto | `#admin reservas del jueves` |
| Ver reservas de una fecha | `#admin reservas del 25/04` |
| Confirmar reserva | `#admin confirmar María` |
| Rechazar reserva | `#admin rechazar Juan sin disponibilidad` |
| Cancelar reserva ya confirmada | `#admin cancelar María` |
| Marcar como no presentado | `#admin no-show Pedro` |
| Cambiar hora | `#admin editar Juan hora 15:30` |
| Cambiar personas | `#admin editar Juan personas 6` |
| Cambiar fecha | `#admin editar Juan fecha 26/04/2026` |
| Añadir nota a reserva | `#admin editar Juan notas sin gluten` |

Cuando confirmas o rechazas, **el cliente recibe el aviso automáticamente por WhatsApp**.

---

## Operativa diaria

| Lo que quieres hacer | Comando |
|---|---|
| Actualizar menú del día | `#admin menú hoy [descripción]` |
| Marcar plato agotado | `#admin agotado bacalao` |
| Abrir/cerrar terraza | `#admin terraza abierta` / `#admin terraza cerrada` |
| Añadir nota visible al agente | `#admin nota terraza cerrada por lluvia` |
| Resumen del día | `#admin resumen` |
| Ver preguntas sin responder | `#admin preguntas` |
| Responder una pregunta | `#admin respuesta abc123 Sí admitimos perros en terraza` |

---

## Contenido para Instagram

```
#admin post hoy
#admin post menú del fin de semana
#admin post terraza en verano
```
El asistente genera 3 ideas de post con emojis y hashtags. Tú eliges la que más te guste.

---

## Ver todos los comandos disponibles

```
#admin ayuda
```

---

## Lo que NO puede hacer el asistente por sí solo

Estas cosas requieren avisar a Eduardo (Profitia):

- Cambiar platos de la carta permanentemente
- Cambiar precios de la carta o del menú del día base
- Cambiar horarios del restaurante
- Actualizar alérgenos
- Añadir nuevas funcionalidades

**Cuándo avisar:** cada vez que cambies la carta de temporada o modifiques horarios. Con una semana de antelación es suficiente.

---

## Si el bot no responde

1. Escríbele un mensaje normal primero para comprobar
2. Si sigue sin responder, contacta con Eduardo

---

*Kamea Gastro Bar — Asistente WhatsApp gestionado por Profitia*
