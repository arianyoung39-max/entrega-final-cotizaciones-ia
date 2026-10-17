# 3. Optimización de costos: matriz de decisión de modelos

**Principio:** el modelo justo para cada tarea. Las tareas mecánicas van a un modelo económico, la redacción que lee el paciente a un modelo de mejor calidad, y lo masivo y no urgente a la API de lotes.

Precios usados (USD por millón de tokens, entrada / salida), del catálogo de modelos de n8n y de la página oficial de Anthropic, consultados el 07-oct-2026. Verificar los vigentes en https://www.anthropic.com/pricing antes de presupuestar.

| Modelo | Entrada | Salida |
|---|---|---|
| Claude Haiku 4.5 | $1 | $5 |
| Claude Sonnet 5.5 | $2 | $10 |
| Claude Opus 5.5 (referencia) | $4 | $20 |

---

## 3.1 Matriz de decisión: qué modelo por tarea y por qué

| Tarea en el flujo | Modelo elegido | Por qué este y no otro | Límite de tokens |
|---|---|---|---|
| **Clasificar** la solicitud contra la lista de precios (salida JSON corta) | **Claude Haiku 4.5** | Tarea mecánica: comparar texto contra una lista y devolver 1 de 3 etiquetas. No necesita razonamiento profundo ni buena redacción. Es el modelo más barato y rápido (1,5 s en la prueba real). | `max_tokens = 600` |
| **Redactar** la respuesta al paciente | **Claude Sonnet 5.5** | Es el texto que lee el cliente: tono cálido, claridad, no prometer resultados, incluir la aclaración legal. Aquí la calidad sí vale la diferencia. | `max_tokens = 3.000` (deja margen al razonamiento interno, ver 3.4) |
| Validar formato, rutas, anti-bucle, “¿texto suficiente?” | **Ninguno (reglas de n8n)** | Si una regla resuelve, no se paga IA. Un correo vacío nunca llega a Claude. | — |
| Reporte mensual de solicitudes no convertidas (propuesto) | **Sonnet 5.5 vía Message Batches API** | Volumen alto y **no urgente**: se envía el viernes y se revisa el lunes, así que tolera la ventana de hasta 24 h del lote a cambio del −50 %. | por solicitud |
| Opus 5.5 | **No se usa** | Costaría ~2,4× más sin mejorar una tarea que Sonnet ya resuelve bien. | — |

---

## 3.2 Costo real por solicitud (medido en la ejecución n8n #14)

| Paso | Tokens entrada | Tokens salida | Costo USD |
|---|---|---|---|
| Haiku · clasificar | 851 | 88 | $0,00129 |
| Sonnet · redactar | 610 | 701 *(429 de razonamiento)* | $0,00823 |
| **Total por cotización enviada** | 1.461 | 789 | **$0,00952** (≈ COP 38 con TRM 4.000) |

## 3.3 Proyección mensual y ahorro estimado

**Supuestos (escritos para poder auditarlos):** 300 correos de cotización al mes (unos 10 al día). El 10 % llega sin texto suficiente y no pasa por IA, así que Haiku corre 270 veces. El 60 % del total está en la lista de precios, así que Sonnet corre 180 veces.

| Escenario | Clasificación | Redacción | Costo mensual | Diferencia vs. elegido |
|---|---|---|---|---|
| **Elegido: Haiku + Sonnet + prefiltro sin IA** | 270 × $0,00129 | 180 × $0,00823 | **$1,83** | — |
| Todo con Sonnet 5.5 | 270 × $0,00258 | 180 × $0,00823 | $2,18 | +19 % |
| Todo con Opus 5.5 | 270 × $0,00516 | 180 × $0,01646 | $4,36 | +138 % |

**Ahorro estimado del diseño elegido:**
- Frente a usar Sonnet en todo: **−16 %** ($0,35 al mes).
- Frente a usar Opus en todo: **−58 %** ($2,53 al mes).

El valor absoluto es bajo porque el volumen de una clínica es pequeño. El principio es el que escala: con 10.000 correos al mes, la misma decisión ahorra unos $84 al mes frente a Opus.

## 3.4 Message Batches (−50 %) para el proceso masivo

Caso propuesto: **reporte mensual** que relee las 300 solicitudes del mes y explica por qué no se convirtieron (motivos, procedimientos más pedidos fuera de la lista).

| | Solicitudes | Tokens por solicitud (estimado) | Costo |
|---|---|---|---|
| API estándar (Sonnet 5.5) | 300 | 900 entrada + 300 salida → $0,0048 | $1,44 |
| **Message Batches** (−50 % en entrada y salida) | 300 | — | **$0,72** |

- **Flujo del lote:** (t0) el viernes a las 22:00 se envía `POST /v1/messages/batches` con un `custom_id` por solicitud. (t1) Ventana de hasta 24 h; la mayoría termina en menos de 1 h. (t2) El sábado se consulta hasta `ended` y se descargan los resultados, ubicados por `custom_id`.
- **Por qué tolera la espera:** la dirección revisa el reporte el lunes; nadie lo necesita en segundos.
- **Por qué NO se usa Batches en las cotizaciones:** el paciente espera respuesta en minutos y hay un humano aprobando. Un lote de hasta 24 h rompería el servicio.

## 3.5 Prompt Caching: evaluado y descartado (por ahora)

- El prefijo fijo de la clasificación (instrucciones + lista de 10 procedimientos) mide ~750 tokens.
- El **mínimo cacheable de Haiku 4.5 es de 4.096 tokens**. Por debajo de ese tamaño, `cache_control` se ignora sin dar error, así que hoy no hay ahorro posible.
- **Cuándo se activaría:** si la lista crece a más de 4.096 tokens (unos 60 procedimientos con sinónimos), el prefijo se cachea y su relectura baja a $0,10 por millón (−90 % solo sobre esa parte).
- El prompt ya quedó preparado para eso: la parte fija va primero y la variable (el correo) al final.
- Los ahorros no se suman (“50 % + 90 %” es un error). Combinados, primero se aplica el caché al prefijo y después el −50 % del lote sobre ese total.

## 3.6 Otras optimizaciones aplicadas
1. **Prefiltro sin IA:** correos de menos de 15 caracteres, respuestas “Re:” y remitentes no-reply no llegan a Claude.
2. **Recorte del mensaje** a 2.000 caracteres: evita que un correo enorme o con hilos citados dispare los tokens.
3. **Max Tokens limitados** en cada nodo de IA.
4. **Hallazgo real:** Sonnet 5.5 gastó 429 de 701 tokens de salida (61 %) en razonamiento interno. Ese gasto es el que más pesa en la cuenta: 180 respuestas × 429 tokens × $10 por millón ≈ $0,77 al mes, el 42 % del total. Siguiente optimización: limitar el esfuerzo de razonamiento en la redacción, que es una tarea simple.
5. **Disparador eficiente:** el Gmail Trigger solo toma correos no leídos con “cotización” en el asunto, uno por ejecución. Los sondeos vacíos no consumen ejecuciones.

## 3.7 Costos fijos de plataforma (para el presupuesto completo)
| Servicio | Plan usado en el proyecto | Riesgo / nota |
|---|---|---|
| n8n Cloud | Plan de la instancia `autonegocio` | Cada correo procesado es 1 ejecución. Revisar el límite mensual del plan |
| Claude | Créditos incluidos en n8n (gateway) para las pruebas | En producción: API key propia de Anthropic (recarga mínima de unos USD 5) |
| Airtable | Gratis | Límite de llamadas a la API por mes en el plan Free; cada solicitud usa entre 5 y 8 llamadas |
| Slack | Prueba (después Free) | Free sirve para el HITL; el historial visible es limitado |
| Gmail | Cuenta personal | En producción: un buzón dedicado de la clínica (Google Workspace) |
