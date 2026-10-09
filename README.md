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

## Evidencia y estado de la entrega

Piloto local del 9 de octubre de 2026:

- Captura Apollo: 28 páginas, 2.751 filas y 2.750 IDs únicos; cobertura parcial documentada.
- Research: una llamada Gemini, 1.033 tokens de entrada y 252 de salida; **1.285 tokens totales**.
- Enriquecimiento aprobado: cuatro perfiles y tres emails marcados `verified` por Apollo. El proveedor no informó el débito efectivo de créditos.
- Cuatro pruebas sintéticas verificaron aislamiento del payload, caché, aprobación/reanudación y conflictos de identidad sin consumir API real.
- Sin escrituras CRM ni compras de teléfono.

**El flujo local fue ejecutado; la calificación comercial completa sigue pendiente.** La búsqueda pública del piloto devolvió páginas generales y no validó los perfiles de LinkedIn. HubSpot está diferido, por lo que historial/reentrada y otros controles comerciales no están completos. Una shortlist A/B/C/X o un perfil enriquecido no equivale a una selección final P1/P2. El sistema conserva esos bloqueos y puede terminar con `pending_evidence` en lugar de declarar éxito completo.

## Relación con las charlas de Flock AI Day 2026

Material revisado: *De usar agentes a construirlos* (Francisco Sempé) y *Flock AI Day 2026 — V2* (Federico Vazquez), del 9/10/2026. Son referencias conceptuales y de challenge; no se identificó una rúbrica formal de puntajes en el texto recuperado.

| Concepto de las charlas | Aplicación y estado en este proyecto |
|---|---|
| LangGraph: estado, nodos y transiciones | Grafo explícito de 13 nodos, estado compartido y pausa/reanudación con SQLite |
| MCP para conectar sistemas | Apollo MCP autorizado localmente; HubSpot diferido |
| El prompt orienta, las herramientas validan y el flujo impone controles | Gemini extrae hechos; código valida evidencia/identidad y bloquea compras sin aprobación |
| Especialistas por API con contexto propio | Research realiza una llamada Gemini; Discovery y Audit son componentes deterministas, no especialistas LLM independientes |
| RAG con fuentes y recuperación pertinente | Índice privado FTS5 por palabras; no implementa embeddings ni generación aumentada con ese índice. Los documentos internos no se envían a Gemini gratuito |
| Keys en variables de entorno | Clave Google local; credenciales y datos excluidos de la entrega GitHub |
| Challenge: spec, instrucciones, conector y publicación | Conector implementado y README preparado. Falta documentar una spec e instrucciones versionables, comprobar entrega reproducible del código y publicar una URL segura |

El entregable actual se presenta como **workflow de prospección con IA y componentes especializados**, sin afirmar que todos los nodos son agentes LLM. La publicación web y la ampliación a varios especialistas por API no se hicieron en esta etapa. No se agregan herramientas ni se cambia la arquitectura solo para reproducir ejemplos de las charlas.

La automatización general de instalación se mantiene pendiente hasta validar el sistema completo.

