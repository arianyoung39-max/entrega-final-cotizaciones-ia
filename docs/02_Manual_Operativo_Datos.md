# 2. Manual operativo de datos

Este documento explica el **cerebro** del sistema (base relacional en Airtable), los **esquemas JSON** que viajan entre integraciones y **cómo se opera** el ecosistema en el día a día.

Base: **Clínica Dental – Cotizaciones IA (Entrega Final)** · ID `appWFhwc5ePm9gC1Q`

---

## 2.1 Esquema de tablas vinculadas

```
Pacientes (1) ──< (N) Solicitudes (N) >── (1) Procedimientos
                         │
                         ├──< (N) Log de Errores
                         │
KPIs (1) ───────────────<┘ (todas las solicitudes y todos los errores se vinculan al panel)
```

Ninguna tabla queda aislada: cada solicitud apunta a su paciente, al procedimiento cotizado, a sus errores y al panel de KPIs. **La vinculación la hace el flujo de n8n**, nunca una persona a mano.

### Tabla `Solicitudes` (tabla central)

| Campo | Tipo | Lo escribe | Para qué sirve |
|---|---|---|---|
| ID | Autonumber | Airtable | Número visible de la solicitud (#1, #2…), trazabilidad |
| Estado | Single select | n8n | Máquina de estados (ver 2.2) |
| Fecha recepción | Fecha y hora | n8n | Cuándo llegó el correo |
| Asunto / Mensaje original | Texto | n8n | Lo que escribió el paciente (recortado a 2.000 caracteres) |
| Gmail Message ID | Texto | n8n | Para responder exactamente a ese correo |
| Gmail Thread ID | Texto | n8n | Para responder en el **mismo hilo** |
| Paciente | Vínculo → Pacientes | n8n | Relación con el paciente |
| Procedimiento | Vínculo → Procedimientos | n8n | Procedimiento de la lista que se cotizó |
| Procedimiento detectado | Texto | IA (Haiku) | Lo que la IA entendió que pide |
| Clasificación IA | Single select | IA (Haiku) | En lista / No en lista / Datos incompletos |
| Razonamiento IA | Texto largo | IA (Haiku) | Explicación en una frase (auditable) |
| Borrador respuesta | Texto largo | IA (Sonnet) | Texto que revisa recepción |
| Decisión humana | Single select | Slack HITL | Aprobado / Rechazado |
| Comentario recepción | Texto largo | n8n | Aprobado, rechazado o sin respuesta en 24 h |
| Fecha respuesta | Fecha y hora | n8n | Cuándo salió el correo |
| Modelos IA usados / Tokens consumidos | Texto / Número | n8n | Control de costos por solicitud |
| Es aprobada · Tuvo decisión · Es enviada | Fórmula (0/1) | Airtable | Insumo de los KPIs |
| Horas hasta respuesta | Fórmula | Airtable | Tiempo de respuesta real |
| Log de Errores · Panel KPI | Vínculos | n8n | Relación con errores y con el dashboard |

### Tabla `Pacientes`
| Campo | Tipo | Nota |
|---|---|---|
| Email (primario) | Email | Clave del *upsert*: si el paciente ya existe no se duplica |
| Nombre | Texto | Nombre que trae Gmail |
| Consentimiento datos | Checkbox | Se marca en la cita de valoración (no se infiere desde un correo) |
| Solicitudes | Vínculo | Historial del paciente |

### Tabla `Procedimientos` (lista de precios = contexto de la IA)
| Campo | Tipo | Nota |
|---|---|---|
| Procedimiento | Texto | Nombre exacto que la IA debe devolver |
| Precio desde | Moneda (COP) | **Único precio que la IA puede usar** |
| Sinónimos | Texto largo | Cómo lo piden los pacientes (“calza”, “brackets”…) |
| Nota para el paciente | Texto largo | Aclaraciones que la IA incluye en la respuesta |
| Activo | Checkbox | Solo los activos se envían a la IA (`{Activo}=1`) |

### Tabla `Log de Errores`
Resumen · Nodo que falló · Mensaje de error · Fecha · ID ejecución n8n · Solicitud (vínculo) · Panel KPI (vínculo) · Resuelto (checkbox).

### Tabla `KPIs` (una sola fila: “Panel general”)
Solicitudes recibidas (count) · Errores registrados (count) · Respuestas enviadas (rollup SUM) · Aprobadas / Revisadas por humano (rollup SUM) · **Tasa de aprobación (%)** · **Tasa de error (%)** · Tiempo promedio de respuesta (h). Se publica como Shared View (ver documento 5).

---

## 2.2 Máquina de estados de una solicitud

```
Pendiente ──► Procesado por IA ──► En revisión humana ──► Aprobado por Humano ──► Enviado
    │                │                                  └──► Rechazado
    │                └──► Derivado a recepción  (No en lista / Datos incompletos)
    └──► Derivado a recepción  (correo sin texto suficiente, sin gastar IA)
Cualquier paso con fallo ──► Error  (+ fila en Log de Errores)
```

Los tres estados que exige la consigna (**Pendiente, Procesado por IA, Aprobado por Humano**) existen y los escribe el flujo.

---

## 2.3 Esquemas JSON de transferencia entre integraciones

Los ejemplos completos y válidos están en la carpeta [`/schemas`](../schemas). Resumen del recorrido del dato:

| # | Tramo | Archivo | Qué viaja |
|---|---|---|---|
| 1 | Gmail → n8n | `01_gmail_trigger_entrada.json` | id, threadId, subject, from, text, date |
| 2 | n8n (normalización) | `02_correo_normalizado.json` | Contrato interno: messageId, threadId, email, nombre, asunto, mensaje, fechaRecepcion |
| 3 | n8n ↔ Claude Haiku | `03_claude_haiku_clasificacion.json` | Entra: lista de precios + **solo el texto** del correo. Sale: JSON con clasificacion, procedimiento_detectado, procedimiento_lista, razonamiento |
| 4 | n8n → Airtable | `04_airtable_solicitud.json` | Las 5 fases de escritura del registro (create + updates por id) |
| 5 | Slack → n8n (HITL) | `05_slack_hitl_respuesta.json` | `data.approved` = true / false / ausente |
| 6 | n8n → Airtable (errores) | `06_log_error.json` | Registro de fallo vinculado a la solicitud |
| 7 | Proceso completo | `07_proceso_completo.json` | Objeto del proceso + array de pasos (modelo relacional de la Pre-Entrega 2) |

**Tipos de datos cuidados:** el precio viaja como número (`650000`) y se formatea solo al mostrarlo (`$650.000`); los filtros comparan booleano con booleano y texto con texto (n8n en modo `strict`); los vínculos de Airtable viajan como **array de IDs**.

---

## 2.4 Cómo se opera (manual del día a día)

| Situación | Qué hace el sistema | Qué hace recepción |
|---|---|---|
| Llega una cotización de un procedimiento de la lista | Registra, clasifica, redacta y **pausa** | Lee el borrador en Slack #recepcion y pulsa **Aprobar y enviar** o **Rechazar** |
| Recepción rechaza o no responde en 24 h | No envía nada, avisa en Slack | Contacta al paciente a mano desde el link del registro |
| Procedimiento fuera de la lista, varios procedimientos o síntomas | Estado *Derivado a recepción* + aviso en Slack | Responde manualmente |
| Correo vacío o de una palabra | No gasta IA; lo deriva | Responde manualmente |
| El paciente responde en el mismo hilo (“Re:”) | **Lo ignora** (anti-bucle) y lo deja sin leer | Lo atiende desde Gmail |
| Falla Claude, Gmail o la IA devuelve basura | Reintenta, registra el error, marca *Error*, alerta en Slack | Revisa el Log y atiende el caso |
| Cambiar un precio o agregar un procedimiento | — | Edita la tabla **Procedimientos** (marcar *Activo*). No hay que tocar n8n |
| Cambiar nombre de la clínica o enlace de agenda | — | Editar el nodo **Configuración del sistema** (único lugar con constantes) |

**Puesta en marcha:** el flujo `Entrega Final – Cotizaciones IA Clínica Dental` debe estar *Published/Active*. El manejador global de errores también debe estar publicado.
