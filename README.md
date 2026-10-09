# Amuyai · Sistema agéntico de calificación de leads

Proyecto integrador del curso de AI Automation Avanzado.
Autor: Rómulo Coronado.
Owner: JPCOAL
Repositorio: amuyai-sistema-agentico

Sistema construido en n8n que atiende el primer contacto comercial de Amuyai, una consultora peruana de automatización con inteligencia artificial para pequeñas y medianas empresas. El agente conversa con el prospecto, entiende qué proceso de su negocio quiere automatizar, lo registra en la base de datos y reporta la operación al equipo.

Este repositorio es acumulativo: el mismo sistema crece módulo a módulo hasta el Proyecto Final Integrador. Cada checkpoint parte del anterior y le suma una capa.

---

## Estado del proyecto

| Módulo | Capa que agrega | Estado |
|---|---|---|
| M1 | Agente base, herramienta y observabilidad | Entregado |
| M2 | Orquestación multi-agente con sub-workflows | Entregado |
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

### Prueba de ejecución manual

La prueba se ejecutó manualmente desde el chat de prueba del workflow en n8n.

| Paso | Detalle |
|---|---|
| Personaje de prueba | Carla Mendoza, Textiles del Sur, carla.mendoza@textilesdelsur.pe |
| Guion | Cuatro mensajes de conversación con el agente |
| Resultado | El agente recogió los cuatro datos y llamó a la herramienta Registrar Lead |
| Verificación en Airtable | Registro creado con ID `recP2L1rHOKEHBYt1` |
| Verificación en Slack | Log enviado a `amuyai-logs` con `intermediateSteps` poblado |
| Auditoría visual | Todos los nodos de la ejecución en verde en n8n |

### Stack

| Tecnología | Rol |
|---|---|
| n8n Cloud | Orquestador |
| OpenAI | Modelo de lenguaje del agente |
| Airtable | Base de datos de leads |
| Slack | Canal de observabilidad interna |

---

## Checkpoint 2 · Orquestación Manager-Worker

**Archivos:**
- [checkpoint2_manager_romulo_coronado.json](checkpoint2_manager_romulo_coronado.json)
- [checkpoint2_worker_registrar_romulo_coronado.json](checkpoint2_worker_registrar_romulo_coronado.json)
- [checkpoint2_worker_consultar_romulo_coronado.json](checkpoint2_worker_consultar_romulo_coronado.json)

El agente único del CP1 se divide en tres workflows independientes: un Manager que clasifica cada mensaje y delega, y dos Workers en lienzos separados, cada uno dedicado a una sola tarea.

### Arquitectura

Manager:

```
[When chat message received]
            ↓
   [Preparar Contexto]
            ↓
        [Router]
    ╎       ╎       ╎
 OpenAI   Simple  Structured
Chat Model Memory Output Parser
            ↓
        [Switch]
            ↓
  NUEVO_LEAD   -> [Payload Registro] -> [Ejecutar Worker Registro] -> [Respuesta Registro]
  SEGUIMIENTO  -> [Tiene Email]
                    sí -> [Payload Consulta] -> [Ejecutar Worker Consulta] -> [Respuesta Consulta]
                    no -> [Pedir Email]
  NO_COMERCIAL -> [Respuesta Directa]
  ESCALAR      -> [Alerta Escalamiento] (Slack) -> [Respuesta Escalamiento]
            ↓
   [Consolidar Salida]
            ↓
   [Log Trazabilidad] (Slack)
            ↓
     [Respuesta Chat]
```

Worker Registrar Lead:

```
[When Executed by Another Workflow]
            ↓
   [Agente Comercial Amuyai]
    ╎          ╎          ╎
 OpenAI     Simple    Registrar Lead
Chat Model  Memory      (Airtable)
            ↓
  Success -> [Salida Registro]
  Error   -> [Salida Error]
```

Worker Consultar Lead:

```
[When Executed by Another Workflow]
            ↓
     [Search records] (Airtable)
            ↓
  Success -> [If]
               true  -> [Salida Encontrado]
               false -> [Salida No Encontrado]
  Error   -> [Salida Error]
```

### Componentes

| Workflow | Nodo | Rol |
|---|---|---|
| Manager | Preparar Contexto | Genera el `trace_id` de la ejecución y normaliza `session_id` y mensaje |
| Manager | Router | Clasifica el mensaje en una taxonomía cerrada de cuatro categorías |
| Manager | Switch | Enruta según la categoría devuelta por el Router |
| Manager | Payload Registro / Payload Consulta | Nodos Set que arman el payload mínimo antes de delegar |
| Manager | Ejecutar Worker Registro / Consulta | Nodos Execute Workflow con Wait for child to finish activado |
| Manager | Consolidar Salida y Log Trazabilidad | Registran en Slack qué Worker se invocó, con qué parámetros y qué devolvió |
| Worker Registrar | Agente Comercial Amuyai | Recoge los datos del prospecto en uno o varios turnos y crea el lead |
| Worker Consultar | Search records + If | Busca el lead por correo y devuelve sus datos o `not_found` |

### Criterio de enrutamiento

| Categoría | Cuándo se asigna | Ruta |
|---|---|---|
| `NUEVO_LEAD` | Quiere automatizar algo, pide información o entrega sus datos de contacto | Worker Registrar Lead |
| `SEGUIMIENTO` | Pregunta explícitamente por una solicitud previa | Worker Consultar Lead, si hay correo |
| `NO_COMERCIAL` | Consulta ajena al negocio | Respuesta directa del Manager |
| `ESCALAR` | Queja, pedido de hablar con una persona o intención dudosa | Alerta a Slack y aviso al usuario |

La taxonomía es cerrada: el Structured Output Parser obliga la categoría con un JSON Schema de tipo `enum`. Ante duda, el Router elige `ESCALAR`, y la salida de respaldo del Switch también apunta a `ESCALAR`.

### Contrato de datos

| Contrato | Campos de entrada |
|---|---|
| Manager a Worker Registrar Lead | `session_id`, `mensaje_usuario`, `trace_id` |
| Manager a Worker Consultar Lead | `email_busqueda`, `trace_id` |

Los dos Workers devuelven siempre un campo `status`: `success`, `pending` (faltan datos del prospecto), `not_found` (el correo no existe) o `error` (fallo técnico, con `message` y `detalle`). Todas las salidas incluyen el `trace_id`.

### Decisiones de diseño

**El Worker Consultar Lead no usa IA.** Buscar un lead por correo es una consulta exacta. Un agente gastaría tokens y agregaría variabilidad sin ningún beneficio.

**Cada delegación pasa por un nodo Set.** El Worker recibe solo los campos que necesita, nunca el payload completo del chat, y el contrato queda explícito.

**El Router tiene memoria por `session_id`.** Si hay un registro en curso, un mensaje que solo trae el correo sigue siendo `NUEVO_LEAD` y no se confunde con un seguimiento.

**Ningún fallo de un Worker detiene al Manager.** Cada Worker reintenta (Retry On Fail) y, si sigue fallando, devuelve un JSON de contingencia con `status: error` por su salida de error.

**El timeout se configura en cada Worker.** Timeout Workflow de 60 segundos en los Settings de cada sub-workflow, porque el nodo Execute Workflow de esta versión de n8n no expone esa opción.

### Prueba de ejecución manual

La prueba se ejecutó manualmente desde el chat de prueba del Manager en n8n, en dos turnos.

| Paso | Detalle |
|---|---|
| Personaje de prueba | Rosa Huamán, Agroexport Valle Verde, rosa.huaman@valleverde.pe |
| Turno 1 (ejecución #26) | "Hola, soy Rosa Huamán de Agroexport Valle Verde. Queremos automatizar el seguimiento de nuestros pedidos de exportación." El Worker devuelve `pending` y el sistema pide el correo |
| Turno 2 (ejecución #28) | "Mi correo es rosa.huaman@valleverde.pe". El Router mantiene `NUEVO_LEAD`, el Worker devuelve `success` con `lead_id` |
| Verificación en Airtable | Registro de Rosa creado con los cuatro datos |
| Verificación en Slack | Log de trazabilidad en `amuyai-logs` con categoría, Worker, parámetros y resultado |
| Auditoría visual | Todos los nodos de la ruta en verde en el panel Executions del Manager |

### Stack

| Tecnología | Rol |
|---|---|
| n8n Cloud | Orquestador del Manager y de los dos Workers |
| OpenAI `gpt-5-mini` | Modelo del Router y del agente de registro, elegido de la lista que ofrece n8n con su credencial Gateway credits |
| Airtable | Base de datos de leads |
| Slack | Log de trazabilidad y alertas de escalamiento |

---

## Sobre las credenciales

Los archivos JSON de este repositorio no contienen claves de API ni tokens. n8n las sustituye por un identificador interno de conexión que solo tiene sentido dentro de la cuenta de origen.

Para reconstruir el sistema hay que importar el JSON en n8n y reconectar cada integración con credenciales propias, además de reapuntar los identificadores de la base de Airtable y del canal de Slack.
