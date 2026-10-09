# flock-agents-iaday

Sistema local de prospección B2B que usa **LangGraph como orquestador**, **Apollo MCP como fuente de contactos** y **Gemini API de Google AI Studio para analizar evidencia pública**. Recibe un comando en español, prepara candidatos, pide autorización antes de enriquecimientos pagos y muestra los resultados en pantalla y JSON.

## Funcionamiento

```text
Comando → políticas locales → cuenta → captura Apollo → identidad/historial
        → Discovery → Research (Gemini) → Audit → propuesta de enriquecimiento
        → aprobación humana → enriquecimiento Apollo → resultados
```

LangGraph organiza 13 nodos sobre un estado compartido: cuenta, candidatos, evidencia, controles pendientes, aprobación y métricas. Define el orden de ejecución y la ruta condicional de enriquecimiento. La aprobación usa una interrupción del grafo; los checkpoints SQLite permiten reanudar la ejecución desde esa pausa.

Los agentes son responsabilidades dentro del flujo; **no todos son llamadas a un modelo**:

| Componente | Responsabilidad | Implementación actual |
|---|---|---|
| Discovery | Reconciliar la captura, aplicar reglas de elegibilidad y ordenar candidatos A/B/C/X sin inventar cupos | Código determinista local |
| Research | Recuperar fuentes públicas, extraer hechos con citas y comprobar su correspondencia con los candidatos | Recuperación local + Gemini 3.1 Flash-Lite |
| Audit | Revisar evidencia y bloqueos antes de admitir prioridades finales P1/P2 | Código determinista local |
| Enrichment | Preparar alcance/costo, pausar para aprobación y recuperar perfil/email corporativo | LangGraph interrupt + Apollo MCP |

**Gemini tiene una única función activa:** extracción estructurada de hechos desde textos públicos. Se invoca mediante `google-genai` y una clave de Google AI Studio; no requiere mantener su interfaz web abierta. No decide compras, no recibe la población completa de contactos y no sustituye las compuertas comerciales locales.

## Ejecución local

Entorno probado: Windows, Python 3.12.10 y LangGraph 1.2.14. El entorno debe tener las dependencias de `pyproject.toml`, autorización Apollo MCP y una clave `GOOGLE_API_KEY` o `GEMINI_API_KEY` configurada localmente.

```powershell
cd C:\AI\projects\flock-agents
.\.venv\Scripts\python.exe flock.py "prospectar Tenaris 4 contactos"
```

La configuración de cuentas y el índice de conocimiento comercial son locales y privados. **Clonar el código no proporciona esos datos ni las credenciales.** Tenaris es la cuenta del piloto. Empresas nuevas se resuelven mediante el lookup gratuito de Apollo; solo se acepta un nombre exacto único con dominio. La captura inicial mantiene el alcance Argentina. Si hay ambigüedad se detiene; ICP e historial permanecen pendientes hasta tener evidencia, sin heredar validaciones del piloto. Se admiten entre 1 y 15 contactos.

La propuesta indica contactos, alcance y límite de créditos. Si la ejecución queda pausada, se puede resolver y consultar el consumo con el ID mostrado:

```powershell
.\.venv\Scripts\python.exe flock.py resume --run-id ID_MOSTRADO --decision approve
.\.venv\Scripts\python.exe flock.py tokens --run-id ID_MOSTRADO
```

También se admite `--decision decline`. Los resultados y evidencias se guardan bajo `.local/flock/runs/`; los checkpoints del grafo, en `.local/flock/graph.sqlite3`. Los contratos/estados internos usan inglés y el informe principal se muestra en español; las evidencias conservan el texto original.

## Inspección con LangSmith Studio

```powershell
.\.venv\Scripts\langgraph.exe dev --host 127.0.0.1 --port 2024 --no-browser --allow-blocking --config langgraph.json
```

Mantener abierta la terminal y entrar a [Studio](https://smith.langchain.com/studio/?baseUrl=http%3A%2F%2F127.0.0.1%3A2024). Elegir el grafo `prospeccion` e ingresar:

```json
{"command":"prospectar Tenaris 4 contactos","mode":"auto"}
```

Studio permite visualizar nodos, estado e interrupciones y ejecutar el grafo. La lógica se modifica en `flock_app/pipeline.py`; algunos cambios requieren reiniciar el servidor. LangGraph orquesta, Gemini analiza y Studio permite inspeccionar. El aviso sobre `LANGSMITH_API_KEY` corresponde a trazas opcionales; las trazas externas están desactivadas en esta configuración.

## Conocimiento, costos y privacidad

Las instrucciones por responsabilidad están en `skills/`. El conocimiento privado se recupera localmente con SQLite FTS5; no usa embeddings externos. No todas las reglas del workflow comercial están automatizadas todavía.

Gemini recibe únicamente textos públicos con procedencia y hash comprobados. El canon, RAG, CRM, historial y respuestas pagas de Apollo permanecen locales. Se limita el texto y se permite como máximo una llamada de modelo por ejecución, con caché y medición de tokens reportados por el proveedor.

El enriquecimiento estándar aprobado comprende perfil y email corporativo, con límite de hasta un crédito base por contacto. No revela teléfonos ni emails personales, no usa waterfall y no escribe ni envía mensajes al CRM. Un registro durable evita recomprar respuestas capturadas; una operación de resultado incierto requiere revisión y no se reintenta automáticamente.

No versionar `.env`, `.local/`, OAuth, bases privadas, fuentes canónicas, capturas, datos de contactos ni ZIP de instalación con conocimiento interno.


