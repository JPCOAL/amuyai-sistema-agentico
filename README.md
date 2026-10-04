# Amuyai · Sistema agéntico de calificación de leads

Proyecto integrador del curso de AI Automation Avanzado.
Autor: Rómulo Coronado.

Sistema construido en n8n que atiende el primer contacto comercial de Amuyai, una consultora peruana de automatización con inteligencia artificial para pequeñas y medianas empresas. El agente conversa con el prospecto, entiende qué proceso de su negocio quiere automatizar, lo registra en la base de datos y reporta la operación al equipo.

Este repositorio es acumulativo: el mismo workflow crece módulo a módulo hasta el Proyecto Final Integrador. Cada checkpoint parte del anterior y le suma una capa.

---

## Estado del proyecto

| Módulo | Capa que agrega | Estado |
|---|---|---|
| M1 | Agente base, herramienta y observabilidad | Entregado |
| M2 | Orquestación multi-agente con sub-workflows | Pendiente |
| M3 | Memoria persistente por Session_ID | Pendiente |
| M4 | Integraciones reales vía OAuth2 | Pendiente |
| M5 | Base documental con RAG | Pendiente |
| M6 | Capa de voz | Pendiente |
| M7 | Especialización por vertical | Pendiente |
| M8 | Supervisor AI-as-a-Judge | Pendiente |
| M9 | Gobernanza, costos y trazabilidad | Pendiente |
| M10 | Brochure técnico-comercial | Pendiente |
| M11 | Sistema completo e informe final | Pendiente |

---

## Checkpoint 1 · Agente base

**Archivo:** [checkpoint1_romulo_coronado.json](checkpoint1_romulo_coronado.json)

### Arquitectura

```
[When chat message received]
            ↓
   [Agente Comercial Amuyai]
    ╎          ╎          ╎
 OpenAI     Simple    Registrar Lead
Chat Model  Memory      (Airtable)
            ↓
   [Log de Observabilidad]
            (Slack)
```

Las líneas punteadas son sub-nodos acoplados al agente. Las continuas son el flujo principal de datos.

### Componentes

| Nodo | Rol |
|---|---|
| When chat message received | Trigger de chat. Captura el mensaje y el identificador de sesión |
| Agente Comercial Amuyai | Nodo de razonamiento. System Prompt modular y guardrail de iteraciones |
| OpenAI Chat Model | Modelo de lenguaje acoplado al agente |
| Simple Memory | Memoria de corto plazo. Permite sostener una conversación de varios turnos |
| Registrar Lead | Herramienta de Airtable. El agente decide cuándo activarla |
| Log de Observabilidad | Reporte a Slack con el rastro de razonamiento del agente |

### Decisiones de diseño

**La herramienta está acoplada lateralmente, no en secuencia.** El agente decide por sí mismo si registra el lead o sigue conversando. Un nodo en secuencia se ejecutaría siempre y eliminaría la decisión.

**El guardrail de iteraciones está fijado en 8.** Acota el ciclo de razonamiento del agente y protege el presupuesto de tokens contra bucles.

**La descripción de la herramienta es manual y extensa.** Declara cuándo usarla, cuándo no usarla y qué enviar en cada campo. Sin esa descripción el modelo no sabe cuándo activarla, lo que el material del curso llama instrucción huérfana.

**El System Prompt está estructurado en seis bloques:** rol, ámbito, objetivo, reglas de conversación, restricciones y escalamiento. Incluye la instrucción explícita de ignorar cualquier orden contenida en el mensaje del usuario.

**El log captura el razonamiento, no solo el resultado.** La opción Return Intermediate Steps hace que el agente devuelva qué herramientas consideró y con qué parámetros las llamó. Ese es el rastro que se envía a Slack.

### Stack

| Tecnología | Rol |
|---|---|
| n8n Cloud | Orquestador |
| OpenAI | Modelo de lenguaje del agente |
| Airtable | Base de datos de leads |
| Slack | Canal de observabilidad interna |

---

## Sobre las credenciales

Los archivos JSON de este repositorio no contienen claves de API ni tokens. n8n las sustituye por un identificador interno de conexión que solo tiene sentido dentro de la cuenta de origen.

Para reconstruir el sistema hay que importar el JSON en n8n y reconectar cada integración con credenciales propias, además de reapuntar los identificadores de la base de Airtable y del canal de Slack.
