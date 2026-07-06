### Entrega Final — Curso AI Automation
**Alumno:** Juan Pablo Groppo

---

## Descripción

Sistema de automatización end-to-end que recibe tickets de soporte técnico, los clasifica con inteligencia artificial (Claude de Anthropic) y los deriva automáticamente por una de tres rutas según su prioridad:

- **Ruta Urgente:** aprobación humana antes de responder al cliente (HITL)
- **Ruta Auto-respuesta:** respuesta automática para consultas simples
- **Ruta Error:** registro y alerta cuando los datos son inválidos o la IA falla

Todo el flujo opera sin intervención manual salvo en el punto de control humano.

---

## Stack Tecnológico

| Componente | Tecnología | Rol |
|---|---|---|
| Orquestador | n8n (local) | Gestiona el flujo completo |
| Base de datos | Airtable | Memoria y registro del sistema |
| Motor de IA | Claude Sonnet 4.6 (Anthropic) | Clasifica tickets y redacta respuestas |
| Canal de salida | Gmail (OAuth2) | Envía respuestas al cliente |
| Notificaciones | Slack (Webhook) | Alertas HITL y errores |
| Trigger | Webhook n8n | Recibe tickets desde formulario externo |

---

## Estructura del Repositorio

```
📁 entrega-final/
├── 📄 README.md
├── 📄 proyecto-coder.json         ← Flujo exportado de n8n
├── 📄 arquitectura-entrega-final.pdf  ← Diagrama de arquitectura
└── 📁 screenshots/
    ├── 01-flujo-n8n.png           ← Flujo completo en n8n
    ├── 02-airtable-tickets.png    ← Registros en Airtable
    ├── 03-slack-aprobacion.png    ← Notificación HITL en Slack
    ├── 04-email-respuesta.png     ← Email enviado al cliente
    └── 05-slack-error.png         ← Alerta de error en Slack
```

---

## Arquitectura del Flujo

```
[Webhook] → [Validar Datos] → [Guardar Ticket en Airtable]
                                        ↓
                              [Analizar con Claude]
                                        ↓
                              [Actualizar Ticket IA]
                                        ↓
                                   [Router]
                          /            |            \
               [Urgente]          [Auto]           [Error]
                  ↓                  ↓                ↓
           [Airtable]           [Gmail]          [Airtable]
           [Slack HITL]         [Airtable]       [Slack Alerta]
           [Wait]
           [Gmail]
           [Airtable]
```

### Nodos del flujo

| # | Nodo | Descripción |
|---|---|---|
| 1 | Recibir Ticket | Webhook POST que recibe nombre, email, asunto y descripción |
| 2 | Validar Datos | IF: verifica que email y descripción no estén vacíos |
| 3 | Guardar Ticket | Crea registro en Airtable con estado "Pendiente" |
| 4 | Analizar con Claude | LLM devuelve categoría, prioridad, resumen y respuesta en JSON |
| 5 | Actualizar Ticket IA | Guarda clasificación de Claude. Estado → "Procesado por IA" |
| 6 | Router | Evalúa prioridad y deriva a la rama correspondiente |

---

## Las Tres Ramas

### Rama Urgente (HITL)
Para tickets de prioridad **Alta** o categoría **Urgente / Facturación**.

1. Airtable → Estado: `Esperando Aprobación`
2. Slack notifica al equipo con resumen y link de aprobación
3. El flujo se **pausa** hasta que un humano aprueba (HITL)
4. Gmail envía la respuesta al cliente
5. Airtable → Estado: `Aprobado`

### Rama Auto-respuesta
Para tickets de prioridad **Media** o **Baja** (consultas simples).

1. Gmail envía respuesta automática al cliente
2. Airtable → Estado: `Auto-respondido`

### Rama Error
Para datos inválidos o fallo de la API de Claude.

1. Airtable → Estado: `Error - Revisión Manual` (si el ticket fue guardado)
2. Slack alerta al equipo
3. Sin contacto al cliente

---

## Estrategia de Resiliencia

| Escenario | Mecanismo | Resultado |
|---|---|---|
| Ticket sin email o descripción | IF (False) → rama error | Alerta Slack, sin contacto al cliente |
| Falla de API de Claude | Continue using error output + Retry On Fail | Registro en Airtable + alerta Slack |
| Prioridad inesperada del LLM | Fallback (Extra Output) del Router | Deriva a rama de error automáticamente |
| Flujo sin aprobación humana | Nodo Wait pausa la ejecución | Email no se envía hasta recibir aprobación |

---

## Base de Datos (Airtable)

🔗 **[Ver base de datos en modo lectura](https://airtable.com/invite/l?inviteId=invUAdc0Txg1wqbFi&inviteToken=2a4451fa2409496efbb8e44d6d1d86e716945a367165dfee70266da0c6938ca1&utm_medium=email&utm_source=product_team&utm_content=transactional-alerts)

### Tabla: Tickets
| Campo | Tipo |
|---|---|
| Nombre | Text |
| Email | Email |
| Asunto | Text |
| Descripcion | Long Text |
| Categoria | Single Select (Bug / Consulta / Urgente / Facturación) |
| Prioridad | Single Select (Alta / Media / Baja) |
| Estado | Single Select (Pendiente / Procesado por IA / Esperando Aprobación / Aprobado / Auto-respondido / Error) |
| Resumen IA | Long Text |
| Respuesta Sugerida | Long Text |
| Fecha | Date |
| Agente | Link to Agentes |

### Tabla: Agentes
| Campo | Tipo |
|---|---|
| Nombre | Text |
| Email | Email |
| Categoria Asignada | Single Select |

---

## Checklist de Requisitos

- [x] Orquestador n8n con flujo principal completo
- [x] Base de datos Airtable con 2 tablas relacionadas y campos de estado
- [x] Motor de IA Claude (Anthropic) con prompt estructurado y variables dinámicas
- [x] Canal de salida Gmail para respuestas al cliente
- [x] Canal de notificación Slack para HITL y alertas de error
- [x] Trigger automático via Webhook (sin intervención manual)
- [x] Human-in-the-Loop con nodo Wait antes de contactar al cliente
- [x] 3 rutas de error: datos inválidos, falla de Claude, fallback del Router
- [x] Error handling con Continue using error output y Retry On Fail
- [x] Registro de error en Airtable cuando falla la API de IA
- [x] Nodos nombrados claramente y sin datos hardcodeados
- [x] 5 pruebas ejecutadas incluyendo camino infeliz

---

## Casos de Prueba

| # | Descripción | Ruta esperada | Resultado |
|---|---|---|---|
| 1 | Consulta simple de horarios | Auto-respuesta | ✅ Email automático enviado |
| 2 | Sistema caído en producción | Urgente + HITL | ✅ Slack notificó, email enviado tras aprobación |
| 3 | Cobro incorrecto en factura | Urgente + HITL | ✅ Slack notificó, email enviado tras aprobación |
| 4 | Bug que cierra la aplicación | Urgente + HITL | ✅ Slack notificó, email enviado tras aprobación |
| 5 | Ticket sin email ni descripción | Error | ✅ Alerta Slack, sin contacto al cliente |

## Video Demo
[Ver demo del sistema] https://youtu.be/zmArV3rlgFE
---

*Curso AI Automation — Juan Pablo Groppo — 2026*
