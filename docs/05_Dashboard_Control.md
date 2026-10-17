# 5. Dashboard de control

**Enlace público (Shared View, abre sin iniciar sesión):**
https://airtable.com/appWFhwc5ePm9gC1Q/shrhSWFJYrNLM9rrK/tblf1nP87oWaD1zGa

Se actualiza solo: cada solicitud y cada error que el flujo registra se vinculan al “Panel general” y los rollups recalculan los KPIs.

| KPI | Fórmula | Qué mide | Valor tras las pruebas |
|---|---|---|---|
| **Tasa de aprobación (%)** | Aprobadas por humano ÷ Revisadas por humano × 100 | Calidad de la IA: cuántos borradores aprueba recepción | 50 % (1 de 2) |
| **Volumen de salida** (Respuestas enviadas) | SUMA de solicitudes en estado *Enviado* | Producción real del sistema | 1 |
| **Tasa de error (%)** | Errores registrados ÷ Solicitudes recibidas × 100 | Estabilidad técnica | 16,7 % (1 de 6) |
| Solicitudes recibidas | Conteo de solicitudes vinculadas | Volumen de entrada | 6 |
| Tiempo promedio de respuesta (h) | Promedio de *Horas hasta respuesta* | Velocidad, incluida la espera humana | 0,13 h |

**Cómo leerlo:**
- Una tasa de aprobación baja significa que hay que ajustar el prompt de Sonnet.
- Una tasa de error alta significa que hay que revisar el Log de Errores (credenciales, modelo, cuotas).
- Mucho volumen de entrada con poca salida significa que la lista de precios no cubre lo que piden los pacientes: hay que agregar procedimientos.

Verificado en una ventana de incógnito (evidencia 07).
