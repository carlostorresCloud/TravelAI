# ✈️ Travel AI

**Proyecto Final — Curso de AI Automation, Coderhouse**
**Alumno:** Carlos Torres Sanchez

Motor de sugerencias de planes de viaje a medida, impulsado por inteligencia artificial en función de las preferencias del viajero. El sistema automatiza el ciclo completo: desde el registro del cliente hasta la generación y validación humana del itinerario final.

---

## 📋 Descripción general

Travel AI integra dos procesos principales:

1. **Registro de perfiles de usuario** en Airtable, a partir de los datos que ingresa un nuevo viajero.
2. **Automatización de generación de propuestas**, disparada al detectar un cambio de estatus de `Pendiente` a `Procesado por IA`. Este flujo despliega dos agentes inteligentes:
   - **Agente 1:** valida los códigos IATA de los aeropuertos contra la base de datos en Airtable.
   - **Agente 2:** realiza búsquedas externas en Google (y consulta la API de vuelos) para construir la propuesta de viaje definitiva.

El resultado pasa por un esquema de **Human in the loop**: se valida en Slack antes de enviarse al cliente por Gmail.

---

## 🏗️ Arquitectura

| Componente | Tecnología | Función |
|---|---|---|
| **Orquestador** | n8n | Administra los flujos de alta de perfiles y activa la creación de propuestas mediante triggers vinculados a la base de datos. |
| **Cerebro (DB)** | Airtable | Almacena el registro de viajeros y las tablas de aeropuertos que consultan los agentes de IA. |
| **Procesamiento (IA)** | OpenAI (GPT-4o-mini / GPT-5) | Modelos que operan como agentes duales: uno valida códigos IATA, el otro obtiene datos actualizados de destinos para sugerir planes de viaje. |
| **Voz (Salida)** | Gmail y Slack | Esquema participativo: se valida el resultado en Slack antes de enviarlo al cliente vía Gmail. |

### Diagramas de flujo(incluidos en el resumen ejecutivo)

- **Flujo de creación de usuarios:** entrada de datos → orquestación en n8n → almacenamiento en Airtable.
- **Flujo principal:** trigger de Airtable → agentes de IA (validación de aeropuertos + búsqueda y armado de itinerario).
- **Proceso Human in the loop:** validación en Slack → envío al cliente vía Gmail.


---

## 🗄️ Estructura de datos en Airtable

### Tabla 1: Clientes
Registra a las personas interesadas en viajar.

| Campo | Tipo |
|---|---|
| Nombre *(primario)* | Identificador del cliente |
| Email | Texto de línea única |
| Teléfono | Texto de línea única |
| Preferencias de viaje | Texto largo |
| Estado | Selector único: `Pendiente` → `Procesado por IA` → `Aprobado por Humano` / `Rechazado` |
| Solicitudes | Registro vinculado a la tabla *Solicitudes* |

### Tabla 2: Solicitudes
Convierte la intención del cliente en una petición de viaje estructurada.

| Campo | Tipo |
|---|---|
| ID Solicitud *(primario)* | Numérico |
| Origen | Texto de línea única |
| Destino | Texto de línea única |
| Fechas | Texto de línea única |
| Itinerario generado | Texto largo (resultado final) |
| Clientes | Registro vinculado a la tabla *Clientes* |
| Preferencias de viaje *(from Clientes)* | Lookup automático desde el cliente vinculado |

### Tabla 3: Aeropuertos
Catálogo de referencia (no transaccional), usado por el primer agente de IA para validar el aeropuerto antes de continuar el flujo.

| Campo |
|---|
| Nombre del Aeropuerto *(primario)* |
| Código IATA |
| Ciudad |
| País |

🔗 **Dashboard ejecutivo (Airtable):** [Ver base](https://airtable.com/appCPOqfBGyLWCYzB/shralv6Ies1xcHOTm)

---

## 🔒 Seguridad y resiliencia del flujo

### Gobernanza de datos
- **Alcance de datos:** solo se solicitan nombre, correo, teléfono y una breve descripción de los requisitos de viaje. No se recolecta información sensible (tarjetas, contraseñas, etc.).
- **Procesamiento y retención:** n8n transforma los datos antes de enviarlos a APIs de terceros (OpenAI), adaptándolos a lo solicitado por los prompts del sistema.
- **Segmentación en Airtable:** la base "Agencia de Viajes" está segmentada en tablas de Clientes, Solicitudes y Aeropuertos. Los agentes de IA solo acceden a los campos necesarios, minimizando la superficie de exposición ante una posible vulneración de credenciales.

### Rutas de contingencia (Error Handlers)
- Se configuraron nodos **Error Trigger** en las etapas críticas del flujo (llamadas a la API de OpenAI y consultas de búsqueda externa).
- Ante cualquier fallo (timeout, error de formato, indisponibilidad del servicio), el flujo no se detiene abruptamente: se envía una notificación automática al canal de Slack **`error-handling`**.

---

## 🤖 Optimización de modelos y recursos

| Modelo / Método | Tarea asignada | Justificación | Impacto en costos |
|---|---|---|---|
| **GPT-5** | Agente de creación de planes de viaje | Se usa junto con la búsqueda de Google para diseñar el itinerario, integrando información de vuelos. Su capacidad de razonamiento autónomo lo hace idóneo para tareas complejas (buscar y analizar información). | Más costoso, pero justificado por la complejidad de la tarea. |
| **GPT-4o-mini** | Validación IATA y uso de la herramienta de Airtable | Modelo más simple, ideal para tareas básicas como resumir texto o extraer datos. | ~90% más económico que la serie GPT-5 (US $0.10 por millón de tokens vs. hasta US $5 por millón). |
| **API de Aviation Stack** | Extraer aerolíneas, números de vuelo y validar rutas origen-destino | Necesaria para consultar información real de vuelos y enviarla al agente generador de planes. | Gratuita en su etapa de prueba (limitada a 100 solicitudes). |

---

## 🚀 Flujo resumido end-to-end

```
Viajero completa formulario
        │
        ▼
   n8n (orquestador)
        │
        ▼
  Airtable → Tabla Clientes (Estado: Pendiente)
        │
        ▼
Cambio de estado → "Procesado por IA" (trigger)
        │
        ├─▶ Agente 1 (GPT-4o-mini): valida código IATA en Airtable
        │
        └─▶ Agente 2 (GPT-5): búsqueda en Google + API de vuelos
                    │
                    ▼
        Itinerario generado → Tabla Solicitudes
                    │
                    ▼
        Validación humana en Slack (Human in the loop)
                    │
                    ▼
        Envío del itinerario final al cliente vía Gmail
```

---

## 🛠️ Stack tecnológico

- [n8n](https://n8n.io/) — Orquestación de flujos de trabajo
- [Airtable](https://airtable.com/) — Base de datos / CRM
- [OpenAI API](https://platform.openai.com/) — Modelos GPT-4o-mini y GPT-5
- [Aviation Stack API](https://aviationstack.com/) — Datos de vuelos
- Slack y Gmail — Validación humana y notificación al cliente

---

## 📄 Licencia

Proyecto desarrollado con fines educativos como entrega final del curso de AI Automation en Coderhouse.
