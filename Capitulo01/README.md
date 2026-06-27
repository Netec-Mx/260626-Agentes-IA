# Análisis de Casos y Definición de Requisitos Agénticos

## 1. Metadatos

| Campo            | Valor                              |
|------------------|------------------------------------|
| **Duración**     | 28 minutos                         |
| **Complejidad**  | Fácil                              |
| **Nivel Bloom**  | Aplicar (Apply)                    |
| **Lab ID**       | 01-00-01                           |
| **Dependencias** | Ninguna (lab independiente)        |
| **Costo API estimado** | < $0.10 USD (uso mínimo de OpenAI para demostración) |

---

## 2. Descripción General

En este laboratorio analizarás tres casos de uso empresariales reales para determinar con criterios técnicos si cada uno justifica una arquitectura agéntica, o si una solución más simple (chatbot o flujo de IA) es suficiente. Aplicarás la taxonomía conceptual de la Lección 1.1 —chatbot, flujo de IA y agente— para clasificar cada escenario, mapearás los cuatro componentes del ciclo agéntico en el caso más adecuado y redactarás un **Documento de Requisitos Agénticos (ARD)** con criterios de aceptación medibles. No se implementa un agente funcional en este lab; el foco es el análisis conceptual aplicado y la documentación formal de requisitos.

---

## 3. Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Distinguir con criterios técnicos las diferencias fundamentales entre un agente de IA, un chatbot conversacional y un flujo de IA determinista.
- [ ] Identificar y mapear los cuatro componentes esenciales de un agente (razonamiento, planificación, acción y observación) en un caso de uso empresarial real.
- [ ] Clasificar agentes según su nivel de sofisticación (reactivo, deliberativo, autónomo) aplicando una rúbrica de evaluación estructurada.
- [ ] Documentar requisitos agénticos formales en un ARD con criterios de aceptación medibles, justificando la elección de arquitectura.

---

## 4. Prerrequisitos

### Conocimientos Previos

| Conocimiento | Nivel requerido |
|---|---|
| Funcionamiento básico de LLMs (prompt → respuesta) | Básico |
| Concepto de APIs REST y llamadas a servicios externos | Básico |
| Python 3.x y Jupyter Notebooks | Básico |
| Haber leído la Lección 1.1 del curso | **Obligatorio** |

### Accesos Requeridos

| Recurso | Detalle |
|---|---|
| API Key de OpenAI | Válida con acceso a `gpt-4o-mini` |
| Conexión a internet | Mínimo 10 Mbps |
| Jupyter Lab instalado | Versión ≥ 4.0 |

---

## 5. Entorno del Laboratorio

### Hardware Recomendado

| Componente | Mínimo |
|---|---|
| CPU | 4 núcleos (Intel i5 8va gen / AMD Ryzen 5) |
| RAM | 8 GB (lab conceptual; sin modelos locales) |
| Almacenamiento | 2 GB libres para entorno virtual y notebooks |

### Software Requerido

| Software | Versión | Propósito |
|---|---|---|
| Python | 3.11.x | Entorno base |
| JupyterLab | ≥ 4.0 | Análisis y documentación |
| openai | 1.40.x | Demostración de capacidades LLM |
| python-dotenv | 1.0.x | Gestión segura de API keys |
| ipywidgets | 8.x | Widgets interactivos en notebook |

### Configuración del Entorno

Ejecuta los siguientes comandos en tu terminal **antes** de abrir Jupyter Lab:

```bash
# 1. Crear entorno virtual dedicado para el Lab 01
python -m venv venv_lab01
```

```bash
# 2. Activar el entorno virtual
# En Windows:
venv_lab01\Scripts\activate
# En macOS / Linux:
source venv_lab01/bin/activate
```

```bash
# 3. Instalar dependencias
pip install --upgrade pip
pip install jupyterlab==4.2.5 openai==1.40.0 python-dotenv==1.0.1 ipywidgets==8.1.3
```

```bash
# 4. Crear archivo de variables de entorno
# Reemplaza TU_API_KEY con tu clave real
echo "OPENAI_API_KEY=TU_API_KEY" > .env
```

```bash
# 5. Crear directorio de trabajo y lanzar Jupyter Lab
mkdir lab_01_analisis_agentes
cd lab_01_analisis_agentes
jupyter lab
```

> ⚠️ **Seguridad**: Nunca compartas tu archivo `.env` ni lo subas a un repositorio. Agrega `.env` a tu `.gitignore` si usas control de versiones.

---

## 6. Instrucciones Paso a Paso

---

### Paso 1: Creación del Notebook de Análisis

**Objetivo**: Crear y estructurar el notebook principal que servirá como espacio de trabajo para todo el lab.

**Instrucciones**:

1. En Jupyter Lab, crea un nuevo notebook haciendo clic en **File → New → Notebook** y selecciona el kernel `Python 3 (ipykernel)`.
2. Renombra el archivo a `lab_01_analisis_agentes.ipynb` (clic derecho sobre el nombre en el panel lateral → Rename).
3. En la primera celda (tipo **Markdown**), pega el siguiente encabezado:

```markdown
# Lab 01-00-01: Análisis de Casos y Definición de Requisitos Agénticos

**Estudiante:** [Tu nombre]  
**Fecha:** [Fecha de hoy]  
**Curso:** Agentes de IA — Módulo 1  

---

## Objetivos del Lab
- Distinguir chatbot, flujo de IA y agente de IA con criterios técnicos.
- Mapear los componentes del ciclo agéntico en un caso real.
- Redactar un ARD (Agentic Requirements Document).
```

4. En la segunda celda (tipo **Code**), pega el bloque de configuración inicial:

```python
# Celda 1: Configuración del entorno
import os
from dotenv import load_dotenv
from openai import OpenAI

# Cargar variables de entorno desde .env
load_dotenv()

# Inicializar cliente de OpenAI
client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

# Verificar que la API key está cargada (sin mostrarla)
api_key = os.getenv("OPENAI_API_KEY")
if api_key and api_key.startswith("sk-"):
    print("✅ API Key cargada correctamente.")
else:
    print("❌ Error: API Key no encontrada. Verifica tu archivo .env")

print(f"📦 OpenAI SDK versión: {__import__('openai').__version__}")
```

5. Ejecuta la celda con `Shift + Enter`.

**Salida Esperada**:

```
✅ API Key cargada correctamente.
📦 OpenAI SDK versión: 1.40.0
```

**Verificación**: Si ves el mensaje de error en lugar del ✅, revisa que el archivo `.env` esté en el mismo directorio donde lanzaste Jupyter Lab y que la clave comience con `sk-`.

---

### Paso 2: Análisis de los Tres Casos de Uso

**Objetivo**: Estudiar los tres escenarios empresariales y clasificar cada uno según el paradigma de IA más adecuado (chatbot, flujo de IA o agente).

**Instrucciones**:

1. Agrega una nueva celda **Markdown** con el título `## Parte 1: Análisis de Casos de Uso`.
2. Agrega una celda **Code** y pega la siguiente estructura de datos que define los tres casos:

```python
# Celda 2: Definición de los tres casos de uso empresariales
casos_de_uso = {
    "caso_A": {
        "nombre": "Asistente de Soporte Técnico",
        "descripcion": (
            "Una empresa de telecomunicaciones quiere automatizar su soporte de primer nivel. "
            "Los clientes envían mensajes describiendo problemas (WiFi lento, factura incorrecta, "
            "configuración de dispositivo). El sistema debe responder con instrucciones claras "
            "basadas en una base de conocimiento estática de 500 artículos. "
            "Si el cliente no resuelve el problema en 3 turnos, se escala a un agente humano. "
            "El 85% de las consultas tienen solución documentada."
        ),
        "volumen_diario": "~2,000 consultas/día",
        "integraciones_requeridas": ["Base de conocimiento (lectura)", "Sistema de tickets (escritura para escalar)"],
        "variabilidad_casos": "Baja — casos bien tipificados",
        "necesidad_decision_autonoma": "Baja — respuestas predefinidas en 85% de casos",
    },
    "caso_B": {
        "nombre": "Agente de Análisis Financiero",
        "descripcion": (
            "Un banco de inversión necesita un sistema que, dado el nombre de una empresa, "
            "recopile automáticamente datos financieros (precio de acción, P/E ratio, noticias recientes, "
            "reportes trimestrales), los analice, detecte anomalías o señales de riesgo, "
            "y genere un informe de due diligence en PDF. "
            "El proceso puede requerir consultar múltiples fuentes en orden variable según los datos encontrados. "
            "Si detecta una anomalía contable, debe profundizar en datos históricos adicionales."
        ),
        "volumen_diario": "~50 análisis/día",
        "integraciones_requeridas": [
            "API de datos financieros (Bloomberg/Yahoo Finance)",
            "Buscador web de noticias",
            "Sistema de generación de PDF",
            "Base de datos histórica interna"
        ],
        "variabilidad_casos": "Alta — cada empresa requiere una ruta de análisis diferente",
        "necesidad_decision_autonoma": "Alta — debe decidir qué fuentes consultar y cuándo profundizar",
    },
    "caso_C": {
        "nombre": "Automatización de Procesos RPA-like",
        "descripcion": (
            "Una aseguradora procesa 300 solicitudes de pólizas diarias. "
            "Cada solicitud sigue exactamente estos pasos: "
            "(1) Extraer datos del formulario PDF, "
            "(2) Validar contra reglas de negocio documentadas, "
            "(3) Calcular prima usando fórmula actuarial fija, "
            "(4) Generar contrato en plantilla estándar, "
            "(5) Enviar email de confirmación. "
            "El proceso es idéntico para el 98% de los casos. "
            "El 2% restante se escala manualmente."
        ),
        "volumen_diario": "~300 solicitudes/día",
        "integraciones_requeridas": [
            "Parser de PDF",
            "Motor de reglas de negocio",
            "Calculadora actuarial",
            "Generador de documentos",
            "Servicio de email"
        ],
        "variabilidad_casos": "Muy baja — proceso 100% predefinido",
        "necesidad_decision_autonoma": "Muy baja — reglas explícitas para todos los pasos",
    }
}

# Mostrar resumen de los casos
for clave, caso in casos_de_uso.items():
    print(f"\n{'='*60}")
    print(f"📋 {clave.upper()}: {caso['nombre']}")
    print(f"{'='*60}")
    print(f"Variabilidad: {caso['variabilidad_casos']}")
    print(f"Decisión autónoma: {caso['necesidad_decision_autonoma']}")
    print(f"Integraciones: {len(caso['integraciones_requeridas'])} sistemas")
```

3. Ejecuta la celda y revisa el resumen de los tres casos.

**Salida Esperada**:

```
============================================================
📋 CASO_A: Asistente de Soporte Técnico
============================================================
Variabilidad: Baja — casos bien tipificados
Decisión autónoma: Baja — respuestas predefinidas en 85% de casos
Integraciones: 2 sistemas

============================================================
📋 CASO_B: Agente de Análisis Financiero
============================================================
Variabilidad: Alta — cada empresa requiere una ruta de análisis diferente
Decisión autónoma: Alta — debe decidir qué fuentes consultar y cuándo profundizar
Integraciones: 4 sistemas

============================================================
📋 CASO_C: Automatización de Procesos RPA-like
============================================================
Variabilidad: Muy baja — proceso 100% predefinido
Decisión autónoma: Muy baja — reglas explícitas para todos los pasos
Integraciones: 5 sistemas
```

**Verificación**: Los tres casos deben imprimirse sin errores. Nota que el Caso B tiene la mayor variabilidad y necesidad de decisión autónoma — esto es una pista importante para el siguiente paso.

---

### Paso 3: Aplicación de la Rúbrica de Clasificación

**Objetivo**: Aplicar una rúbrica estructurada para clasificar objetivamente cada caso según el paradigma de IA más adecuado.

**Instrucciones**:

1. Agrega una celda **Markdown**: `## Parte 2: Rúbrica de Clasificación de Paradigmas`.
2. Agrega la siguiente celda **Code** con la rúbrica de evaluación:

```python
# Celda 3: Rúbrica de clasificación — basada en la tabla comparativa de la Lección 1.1
rubrica = {
    "criterios": [
        "control_flujo",        # ¿Quién controla el flujo? (predefinido vs dinámico)
        "uso_herramientas",     # ¿Cómo se usan las herramientas? (fijo vs selección dinámica)
        "adaptacion_fallos",    # ¿Se adapta ante resultados inesperados?
        "horizonte_temporal",   # ¿Cuántos ciclos de razonamiento requiere?
        "entrada_objetivo",     # ¿La entrada es un mensaje, datos o un objetivo de alto nivel?
    ],
    "puntuacion": {
        # 1 = Chatbot, 2 = Flujo de IA, 3 = Agente de IA
        "control_flujo":     {"predefinido_lineal": 1, "predefinido_ramificado": 2, "dinamico_llm": 3},
        "uso_herramientas":  {"ninguno_o_limitado": 1, "fijo_en_diseno": 2, "seleccion_dinamica": 3},
        "adaptacion_fallos": {"no_adapta": 1, "reintentos_simples": 2, "rutas_alternativas": 3},
        "horizonte_temporal":{"un_turno": 1, "pipeline_fijo": 2, "multiples_ciclos": 3},
        "entrada_objetivo":  {"mensaje_usuario": 1, "datos_evento": 2, "objetivo_alto_nivel": 3},
    }
}

# Evaluación manual de cada caso según la rúbrica
evaluaciones = {
    "caso_A": {
        "control_flujo": "predefinido_ramificado",   # árbol de diálogo con escalado
        "uso_herramientas": "fijo_en_diseno",         # siempre KB + tickets
        "adaptacion_fallos": "reintentos_simples",    # máximo 3 turnos
        "horizonte_temporal": "pipeline_fijo",         # 3 turnos máximo predefinidos
        "entrada_objetivo": "mensaje_usuario",         # el usuario envía un mensaje
        "justificacion": (
            "El 85% de los casos tiene solución documentada y el flujo está bien tipificado. "
            "Un chatbot avanzado con RAG sobre la base de conocimiento + regla de escalado es suficiente. "
            "No se requiere razonamiento autónomo ni selección dinámica de herramientas."
        )
    },
    "caso_B": {
        "control_flujo": "dinamico_llm",              # el orden de consulta varía por caso
        "uso_herramientas": "seleccion_dinamica",      # decide qué APIs consultar
        "adaptacion_fallos": "rutas_alternativas",    # si detecta anomalía, profundiza
        "horizonte_temporal": "multiples_ciclos",      # ciclos de análisis iterativo
        "entrada_objetivo": "objetivo_alto_nivel",     # "analiza la empresa X"
        "justificacion": (
            "Cada análisis requiere una ruta diferente según los datos encontrados. "
            "El sistema debe decidir cuándo profundizar, qué fuentes priorizar y cómo "
            "estructurar el informe final. Esto requiere razonamiento autónomo: arquitectura agéntica."
        )
    },
    "caso_C": {
        "control_flujo": "predefinido_lineal",        # 5 pasos siempre en el mismo orden
        "uso_herramientas": "fijo_en_diseno",          # herramientas fijas y ordenadas
        "adaptacion_fallos": "no_adapta",              # el 2% se escala manualmente
        "horizonte_temporal": "pipeline_fijo",          # pipeline de 5 pasos
        "entrada_objetivo": "datos_evento",             # formulario PDF como entrada
        "justificacion": (
            "El proceso es 100% predefinido con reglas explícitas. "
            "Un flujo de IA (pipeline) con LLM para extracción de PDF y generación de texto "
            "es suficiente y más eficiente. Usar un agente aquí añadiría complejidad innecesaria, "
            "mayor latencia y mayor costo sin beneficio real."
        )
    }
}

# Calcular puntuaciones
print("📊 RESULTADOS DE LA RÚBRICA DE CLASIFICACIÓN")
print("=" * 65)

paradigma_labels = {1: "🗨️  Chatbot", 2: "⚙️  Flujo de IA", 3: "🤖 Agente de IA"}

for caso_id, eval_data in evaluaciones.items():
    nombre = casos_de_uso[caso_id]["nombre"]
    puntuaciones = []
    
    for criterio in rubrica["criterios"]:
        valor_elegido = eval_data[criterio]
        puntuacion = rubrica["puntuacion"][criterio][valor_elegido]
        puntuaciones.append(puntuacion)
    
    promedio = sum(puntuaciones) / len(puntuaciones)
    clasificacion = round(promedio)
    
    print(f"\n{caso_id.upper()}: {nombre}")
    print(f"  Puntuaciones por criterio: {puntuaciones}")
    print(f"  Promedio: {promedio:.1f} → Clasificación: {paradigma_labels[clasificacion]}")
    print(f"  Justificación: {eval_data['justificacion'][:80]}...")

print("\n" + "=" * 65)
print("✅ Clasificación completada.")
```

3. Ejecuta la celda.

**Salida Esperada**:

```
📊 RESULTADOS DE LA RÚBRICA DE CLASIFICACIÓN
=================================================================

CASO_A: Asistente de Soporte Técnico
  Puntuaciones por criterio: [2, 2, 2, 2, 1]
  Promedio: 1.8 → Clasificación: 🗨️  Chatbot
  Justificación: El 85% de los casos tiene solución documentada y el flujo está...

CASO_B: Agente de Análisis Financiero
  Puntuaciones por criterio: [3, 3, 3, 3, 3]
  Promedio: 3.0 → Clasificación: 🤖 Agente de IA
  Justificación: Cada análisis requiere una ruta diferente según los datos encontrados...

CASO_C: Automatización de Procesos RPA-like
  Puntuaciones por criterio: [1, 2, 1, 2, 2]
  Promedio: 1.6 → Clasificación: ⚙️  Flujo de IA
  Justificación: El proceso es 100% predefinido con reglas explícitas...

=================================================================
✅ Clasificación completada.
```

**Verificación**: El Caso B debe clasificarse como "Agente de IA" con puntuación 3.0. Si algún caso tiene una clasificación diferente a la esperada, revisa las asignaciones de criterios en el diccionario `evaluaciones`.

---

### Paso 4: Mapeo del Ciclo Agéntico en el Caso B

**Objetivo**: Para el caso que justifica arquitectura agéntica (Caso B), identificar y documentar cómo se manifiestan los cuatro componentes esenciales del ciclo agéntico.

**Instrucciones**:

1. Agrega una celda **Markdown**: `## Parte 3: Mapeo del Ciclo Agéntico — Caso B`.
2. Agrega la siguiente celda **Code**:

```python
# Celda 4: Mapeo de los 4 componentes del ciclo agéntico para el Caso B
ciclo_agéntico_caso_B = {
    "caso": "Agente de Análisis Financiero (Caso B)",
    "objetivo_ejemplo": "Genera un informe de due diligence para la empresa 'TechCorp Inc.'",
    
    "componentes": {
        
        "1_razonamiento": {
            "descripcion": "¿Qué hace el agente en esta fase?",
            "manifestacion_caso_B": (
                "El LLM recibe el nombre de la empresa y razona sobre qué información necesita "
                "para un due diligence completo. Evalúa el contexto de los datos ya recopilados "
                "y decide si son suficientes o si debe buscar más. Si encuentra una anomalía "
                "contable, razona sobre su significancia antes de decidir profundizar."
            ),
            "tecnologia_implementacion": "LLM (GPT-4o-mini) con chain-of-thought prompting",
            "pregunta_clave": "¿Qué sé? ¿Qué me falta? ¿Qué debo hacer a continuación?",
        },
        
        "2_planificacion": {
            "descripcion": "¿Cómo organiza el agente sus acciones?",
            "manifestacion_caso_B": (
                "Antes de ejecutar, el agente crea un plan de análisis: "
                "(1) Obtener datos básicos de la acción, "
                "(2) Buscar noticias recientes, "
                "(3) Descargar último reporte trimestral, "
                "(4) Calcular ratios financieros clave, "
                "(5) Detectar anomalías. "
                "Este plan puede modificarse: si en el paso 4 detecta P/E anómalo, "
                "inserta un paso adicional de análisis histórico."
            ),
            "tecnologia_implementacion": "Patrón ReAct / Planner en LangGraph",
            "pregunta_clave": "¿En qué orden ejecuto las acciones? ¿Debo replantear el plan?",
        },
        
        "3_accion": {
            "descripcion": "¿Qué herramientas ejecuta el agente?",
            "manifestacion_caso_B": (
                "El agente invoca herramientas concretas: "
                "- get_stock_data(ticker='TECH') → datos de precio y ratios, "
                "- search_news(company='TechCorp', days=30) → noticias recientes, "
                "- download_quarterly_report(company='TechCorp', quarter='Q3-2024') → PDF, "
                "- analyze_financial_anomalies(data=...) → detección de irregularidades, "
                "- generate_pdf_report(content=...) → informe final."
            ),
            "tecnologia_implementacion": "Tool calling / Function calling de OpenAI",
            "pregunta_clave": "¿Qué herramienta uso? ¿Con qué parámetros la invoco?",
        },
        
        "4_observacion": {
            "descripcion": "¿Cómo evalúa el agente los resultados?",
            "manifestacion_caso_B": (
                "Tras cada acción, el agente observa el resultado y lo integra en su contexto: "
                "- Si get_stock_data() falla, intenta con un ticker alternativo o marca el dato como no disponible. "
                "- Si search_news() devuelve artículos sobre litigios, actualiza su plan para investigar más. "
                "- Si el PDF del reporte no está disponible, busca en fuentes alternativas. "
                "La observación alimenta el siguiente ciclo de razonamiento."
            ),
            "tecnologia_implementacion": "Estado del grafo en LangGraph / memoria de contexto",
            "pregunta_clave": "¿El resultado es lo que esperaba? ¿Debo ajustar el plan?",
        }
    }
}

# Visualizar el mapeo
print(f"🔄 CICLO AGÉNTICO: {ciclo_agéntico_caso_B['caso']}")
print(f"📌 Objetivo de ejemplo: {ciclo_agéntico_caso_B['objetivo_ejemplo']}")
print("=" * 70)

for comp_id, comp_data in ciclo_agéntico_caso_B["componentes"].items():
    nombre_componente = comp_id.replace("_", " ").upper()
    print(f"\n{'─'*70}")
    print(f"  {nombre_componente}")
    print(f"{'─'*70}")
    print(f"  📝 Descripción:      {comp_data['descripcion']}")
    print(f"  🏢 En el Caso B:     {comp_data['manifestacion_caso_B'][:120]}...")
    print(f"  🛠️  Tecnología:       {comp_data['tecnologia_implementacion']}")
    print(f"  ❓ Pregunta clave:   {comp_data['pregunta_clave']}")

print(f"\n{'='*70}")
print("✅ Ciclo agéntico mapeado correctamente.")
```

3. Ejecuta la celda.

**Salida Esperada**: El notebook debe imprimir los cuatro componentes (Razonamiento, Planificación, Acción, Observación) con sus manifestaciones específicas para el Caso B, cada uno con su tecnología de implementación y pregunta clave.

**Verificación**: Confirma que los cuatro componentes están presentes y que cada uno tiene una manifestación concreta relacionada con el análisis financiero (no descripciones genéricas).

---

### Paso 5: Demostración de Razonamiento con OpenAI API

**Objetivo**: Usar la API de OpenAI para demostrar, con un ejemplo mínimo, cómo un LLM puede razonar sobre el Caso B y proponer un plan de acción — ilustrando la diferencia entre una respuesta de chatbot y el razonamiento de un agente.

**Instrucciones**:

1. Agrega una celda **Markdown**: `## Parte 4: Demostración de Razonamiento Agéntico con LLM`.
2. Agrega la siguiente celda **Code**:

```python
# Celda 5: Demostración de razonamiento agéntico vs respuesta de chatbot
# NOTA: Esta celda consume ~$0.02 USD de la API de OpenAI

def demo_chatbot_response(empresa: str) -> str:
    """Simula cómo respondería un chatbot simple al mismo input."""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": "Eres un asistente conversacional. Responde de forma breve y amigable."
            },
            {
                "role": "user",
                "content": f"Necesito información sobre {empresa} para una inversión."
            }
        ],
        max_tokens=150,
        temperature=0.7
    )
    return response.choices[0].message.content


def demo_agente_reasoning(empresa: str) -> str:
    """Demuestra cómo un agente razona y planifica antes de actuar."""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": (
                    "Eres un agente de análisis financiero. Cuando recibes un objetivo, "
                    "PRIMERO razonas sobre qué información necesitas, LUEGO creas un plan "
                    "detallado de las herramientas que usarías (en orden), y FINALMENTE "
                    "explicas cómo evaluarías los resultados para ajustar tu análisis. "
                    "Sé específico sobre las herramientas: get_stock_data(), search_news(), "
                    "download_report(), detect_anomalies(), generate_pdf_report(). "
                    "Estructura tu respuesta en tres secciones: RAZONAMIENTO, PLAN DE ACCIÓN, "
                    "CRITERIO DE OBSERVACIÓN."
                )
            },
            {
                "role": "user",
                "content": (
                    f"Objetivo: Genera un informe completo de due diligence para '{empresa}'. "
                    f"El cliente necesita una decisión de inversión en las próximas 2 horas."
                )
            }
        ],
        max_tokens=500,
        temperature=0.3  # Menor temperatura para razonamiento más estructurado
    )
    return response.choices[0].message.content


# Ejecutar ambas demos
empresa_ejemplo = "TechCorp Inc."

print("=" * 70)
print("🗨️  RESPUESTA DE CHATBOT (paradigma conversacional)")
print("=" * 70)
respuesta_chatbot = demo_chatbot_response(empresa_ejemplo)
print(respuesta_chatbot)

print("\n" + "=" * 70)
print("🤖 RAZONAMIENTO DE AGENTE (paradigma agéntico)")
print("=" * 70)
respuesta_agente = demo_agente_reasoning(empresa_ejemplo)
print(respuesta_agente)

print("\n" + "─" * 70)
print("💡 OBSERVACIÓN CLAVE:")
print("   El chatbot responde con información general sin plan de acción.")
print("   El agente razona, planifica herramientas específicas y define")
print("   criterios para evaluar y ajustar su análisis — ciclo completo.")
```

3. Ejecuta la celda. **Esto hará una llamada real a la API de OpenAI.**

**Salida Esperada**: Dos bloques de texto claramente diferenciados. El chatbot dará una respuesta genérica y conversacional. El agente estructurará su respuesta en secciones de Razonamiento, Plan de Acción y Criterio de Observación, mencionando herramientas específicas.

**Verificación**: La respuesta del agente debe contener al menos dos de las herramientas mencionadas en el prompt del sistema (`get_stock_data`, `search_news`, etc.) y debe estar organizada en secciones. Si la API devuelve un error, verifica tu API key y que tienes crédito disponible.

---

### Paso 6: Redacción del ARD (Agentic Requirements Document)

**Objetivo**: Documentar formalmente los requisitos agénticos del Caso B en un ARD estructurado con criterios de aceptación medibles.

**Instrucciones**:

1. Agrega una celda **Markdown**: `## Parte 5: Documento de Requisitos Agénticos (ARD)`.
2. Agrega la siguiente celda **Code** que genera y valida el ARD:

```python
# Celda 6: Generación y validación del ARD para el Caso B
ard = {
    "metadata": {
        "ard_id": "ARD-001",
        "version": "1.0",
        "caso_uso": "Agente de Análisis Financiero",
        "autor": "[Tu nombre]",
        "fecha": "2024-01-01",  # Actualiza con la fecha actual
        "estado": "BORRADOR",
    },
    
    "justificacion_arquitectura_agéntica": {
        "decision": "AGENTE DE IA",
        "alternativas_descartadas": {
            "chatbot": "Descartado: no puede ejecutar análisis multi-fuente ni generar PDFs.",
            "flujo_ia": "Descartado: el orden de consulta varía por caso; no es predefinible.",
        },
        "criterios_que_justifican_agente": [
            "Control de flujo dinámico: el LLM debe decidir qué fuentes consultar según hallazgos.",
            "Selección dinámica de herramientas: 4+ herramientas con uso condicional.",
            "Adaptación ante fallos: rutas alternativas si una fuente no está disponible.",
            "Horizonte multi-ciclo: el análisis requiere múltiples iteraciones razonamiento→acción.",
            "Entrada de objetivo: el usuario define el objetivo, no los pasos.",
        ]
    },
    
    "componentes_agénticos": {
        "razonamiento": {
            "descripcion": "LLM evalúa datos disponibles y decide próxima acción.",
            "modelo": "gpt-4o-mini (evaluación) / gpt-4o (producción)",
            "patron": "Chain-of-thought con herramientas disponibles en contexto",
        },
        "planificacion": {
            "descripcion": "Generación de plan de análisis dinámico, modificable en ejecución.",
            "patron_arquitectonico": "ReAct (Reasoning + Acting)",
            "max_pasos_plan": 10,
        },
        "accion": {
            "herramientas": [
                {"nombre": "get_stock_data", "tipo": "API externa", "autenticacion": "API Key"},
                {"nombre": "search_news", "tipo": "Web search", "autenticacion": "API Key"},
                {"nombre": "download_quarterly_report", "tipo": "Web scraping / API", "autenticacion": "OAuth"},
                {"nombre": "detect_anomalies", "tipo": "Función interna", "autenticacion": "N/A"},
                {"nombre": "generate_pdf_report", "tipo": "Servicio interno", "autenticacion": "Service token"},
            ]
        },
        "observacion": {
            "descripcion": "Evaluación de resultados de cada herramienta; actualización del estado del agente.",
            "estado_persistido": ["datos_recopilados", "anomalias_detectadas", "plan_actual", "iteracion_actual"],
        }
    },
    
    "nivel_sofisticacion": {
        "clasificacion": "DELIBERATIVO",
        "justificacion": (
            "El agente crea un plan antes de actuar y lo modifica según observaciones. "
            "No es puramente reactivo (no responde solo al estímulo inmediato) pero tampoco "
            "completamente autónomo (opera dentro de un objetivo definido por el usuario y "
            "herramientas predefinidas). Clasificación: Deliberativo."
        ),
        "escala_referencia": {
            "reactivo": "Responde a estímulos inmediatos sin planificación.",
            "deliberativo": "Planifica antes de actuar; modifica el plan según observaciones. ← ESTE CASO",
            "autonomo": "Define sus propios objetivos; opera indefinidamente sin intervención humana.",
        }
    },
    
    "criterios_de_aceptacion": [
        {
            "id": "CA-001",
            "descripcion": "El agente genera un informe de due diligence completo en < 120 segundos.",
            "metrica": "Tiempo de ejecución end-to-end",
            "umbral": "≤ 120 segundos para el 95% de los casos",
            "metodo_verificacion": "Test de rendimiento con 50 empresas del S&P 500",
        },
        {
            "id": "CA-002",
            "descripcion": "El agente consulta al menos 3 fuentes de datos independientes por análisis.",
            "metrica": "Número de herramientas únicas invocadas por ejecución",
            "umbral": "≥ 3 herramientas diferentes en el 100% de los casos",
            "metodo_verificacion": "Inspección de logs de ejecución del agente",
        },
        {
            "id": "CA-003",
            "descripcion": "El agente se recupera de fallos de herramientas individuales sin interrumpir el análisis.",
            "metrica": "Tasa de completitud ante fallo de una herramienta",
            "umbral": "≥ 90% de análisis completados cuando una herramienta falla",
            "metodo_verificacion": "Test de resiliencia inyectando fallos simulados en cada herramienta",
        },
        {
            "id": "CA-004",
            "descripcion": "El informe generado incluye sección de anomalías con evidencia cuando se detectan.",
            "metrica": "Precisión en detección de anomalías contables conocidas",
            "umbral": "≥ 80% de recall en dataset de prueba con anomalías etiquetadas",
            "metodo_verificacion": "Evaluación con dataset de 20 empresas con anomalías documentadas",
        },
        {
            "id": "CA-005",
            "descripcion": "El costo por análisis no supera el umbral operacional.",
            "metrica": "Costo de tokens de LLM por ejecución",
            "umbral": "≤ $0.15 USD por análisis usando gpt-4o-mini",
            "metodo_verificacion": "Monitoreo de uso en LangSmith / OpenAI dashboard",
        },
    ],
    
    "restricciones_y_riesgos": [
        {
            "tipo": "SEGURIDAD",
            "descripcion": "El agente no debe ejecutar acciones de compra/venta. Solo lectura y análisis.",
            "mitigacion": "Herramientas de escritura deshabilitadas; revisión humana obligatoria del informe."
        },
        {
            "tipo": "COSTO",
            "descripcion": "Bucles infinitos de razonamiento pueden generar costos no controlados.",
            "mitigacion": "Límite máximo de 10 iteraciones del ciclo agéntico por ejecución."
        },
        {
            "tipo": "CALIDAD",
            "descripcion": "Datos financieros de APIs externas pueden estar desactualizados.",
            "mitigacion": "El agente debe registrar la fecha/hora de cada dato en el informe."
        }
    ]
}

# Validar que el ARD está completo
secciones_requeridas = [
    "metadata", "justificacion_arquitectura_agéntica", "componentes_agénticos",
    "nivel_sofisticacion", "criterios_de_aceptacion", "restricciones_y_riesgos"
]

print("📄 VALIDACIÓN DEL ARD")
print("=" * 60)
todas_presentes = True
for seccion in secciones_requeridas:
    presente = seccion in ard
    estado = "✅" if presente else "❌"
    print(f"  {estado} {seccion}")
    if not presente:
        todas_presentes = False

print(f"\n  Criterios de aceptación definidos: {len(ard['criterios_de_aceptacion'])}/5")
print(f"  Herramientas documentadas: {len(ard['componentes_agénticos']['accion']['herramientas'])}")
print(f"  Restricciones identificadas: {len(ard['restricciones_y_riesgos'])}")

if todas_presentes and len(ard['criterios_de_aceptacion']) >= 3:
    print("\n✅ ARD VÁLIDO: Todas las secciones requeridas están presentes.")
    print(f"   ARD ID: {ard['metadata']['ard_id']} | Estado: {ard['metadata']['estado']}")
else:
    print("\n❌ ARD INCOMPLETO: Revisa las secciones marcadas con ❌")
```

3. Ejecuta la celda.

**Salida Esperada**:

```
📄 VALIDACIÓN DEL ARD
============================================================
  ✅ metadata
  ✅ justificacion_arquitectura_agéntica
  ✅ componentes_agénticos
  ✅ nivel_sofisticacion
  ✅ criterios_de_aceptacion
  ✅ restricciones_y_riesgos

  Criterios de aceptación definidos: 5/5
  Herramientas documentadas: 5
  Restricciones identificadas: 3

✅ ARD VÁLIDO: Todas las secciones requeridas están presentes.
   ARD ID: ARD-001 | Estado: BORRADOR
```

**Verificación**: El ARD debe validarse como completo (todos los ✅). Actualiza el campo `autor` y `fecha` en `metadata` con tus datos reales.

---

### Paso 7: Exportación del ARD a Markdown

**Objetivo**: Generar el ARD como un documento Markdown independiente para entrega formal.

**Instrucciones**:

1. Agrega una celda **Code** final:

```python
# Celda 7: Exportar el ARD a un archivo Markdown independiente
import json
from datetime import date

def exportar_ard_markdown(ard: dict, filepath: str) -> None:
    """Genera un archivo Markdown del ARD para entrega formal."""
    
    md_lines = []
    
    # Encabezado
    md_lines.append(f"# {ard['metadata']['ard_id']}: Agentic Requirements Document")
    md_lines.append(f"\n**Caso de Uso:** {ard['metadata']['caso_uso']}  ")
    md_lines.append(f"**Versión:** {ard['metadata']['version']}  ")
    md_lines.append(f"**Autor:** {ard['metadata']['autor']}  ")
    md_lines.append(f"**Fecha:** {ard['metadata']['fecha']}  ")
    md_lines.append(f"**Estado:** {ard['metadata']['estado']}  \n")
    md_lines.append("---\n")
    
    # Decisión de arquitectura
    md_lines.append("## 1. Decisión de Arquitectura\n")
    just = ard['justificacion_arquitectura_agéntica']
    md_lines.append(f"**Arquitectura seleccionada:** `{just['decision']}`\n")
    md_lines.append("### Alternativas descartadas\n")
    for alt, razon in just['alternativas_descartadas'].items():
        md_lines.append(f"- **{alt.capitalize()}**: {razon}")
    md_lines.append("\n### Criterios que justifican el uso de agente\n")
    for criterio in just['criterios_que_justifican_agente']:
        md_lines.append(f"- {criterio}")
    md_lines.append("")
    
    # Nivel de sofisticación
    md_lines.append("\n## 2. Nivel de Sofisticación\n")
    sof = ard['nivel_sofisticacion']
    md_lines.append(f"**Clasificación:** `{sof['clasificacion']}`\n")
    md_lines.append(f"{sof['justificacion']}\n")
    
    # Componentes agénticos
    md_lines.append("\n## 3. Componentes del Ciclo Agéntico\n")
    comp = ard['componentes_agénticos']
    md_lines.append(f"### Razonamiento\n{comp['razonamiento']['descripcion']}  ")
    md_lines.append(f"Modelo: `{comp['razonamiento']['modelo']}`\n")
    md_lines.append(f"### Planificación\n{comp['planificacion']['descripcion']}  ")
    md_lines.append(f"Patrón: `{comp['planificacion']['patron_arquitectonico']}`  ")
    md_lines.append(f"Máx. pasos: {comp['planificacion']['max_pasos_plan']}\n")
    md_lines.append("### Herramientas (Acción)\n")
    md_lines.append("| Herramienta | Tipo | Autenticación |")
    md_lines.append("|---|---|---|")
    for h in comp['accion']['herramientas']:
        md_lines.append(f"| `{h['nombre']}` | {h['tipo']} | {h['autenticacion']} |")
    md_lines.append(f"\n### Observación\n{comp['observacion']['descripcion']}\n")
    
    # Criterios de aceptación
    md_lines.append("\n## 4. Criterios de Aceptación\n")
    md_lines.append("| ID | Descripción | Umbral | Método de Verificación |")
    md_lines.append("|---|---|---|---|")
    for ca in ard['criterios_de_aceptacion']:
        md_lines.append(f"| {ca['id']} | {ca['descripcion']} | {ca['umbral']} | {ca['metodo_verificacion']} |")
    
    # Restricciones
    md_lines.append("\n\n## 5. Restricciones y Riesgos\n")
    for r in ard['restricciones_y_riesgos']:
        md_lines.append(f"### [{r['tipo']}] ")
        md_lines.append(f"**Descripción:** {r['descripcion']}  ")
        md_lines.append(f"**Mitigación:** {r['mitigacion']}\n")
    
    # Escribir archivo
    with open(filepath, 'w', encoding='utf-8') as f:
        f.write('\n'.join(md_lines))
    
    print(f"✅ ARD exportado exitosamente a: {filepath}")
    print(f"   Tamaño del archivo: {len(''.join(md_lines))} caracteres")


# Exportar el ARD
nombre_archivo = f"ARD-001_analisis_financiero_{date.today().strftime('%Y%m%d')}.md"
exportar_ard_markdown(ard, nombre_archivo)

# Verificar que el archivo existe
if os.path.exists(nombre_archivo):
    print(f"   📁 Archivo verificado en el directorio de trabajo.")
else:
    print(f"❌ Error: El archivo no se creó correctamente.")
```

2. Ejecuta la celda.

**Salida Esperada**:

```
✅ ARD exportado exitosamente a: ARD-001_analisis_financiero_20240101.md
   Tamaño del archivo: ~3500 caracteres
   📁 Archivo verificado en el directorio de trabajo.
```

**Verificación**: Abre el archivo `.md` generado desde el panel lateral de Jupyter Lab (clic derecho → Open With → Markdown Preview) y confirma que el documento se renderiza correctamente con todas las secciones.

---

## 7. Validación y Pruebas

Ejecuta las siguientes verificaciones para confirmar que el lab está completo:

```python
# Celda de validación final — ejecutar al terminar el lab
print("🔍 VALIDACIÓN FINAL DEL LAB 01-00-01")
print("=" * 60)

validaciones = {
    "API Key configurada": api_key is not None and api_key.startswith("sk-"),
    "Tres casos analizados": len(casos_de_uso) == 3,
    "Rúbrica aplicada a todos los casos": len(evaluaciones) == 3,
    "Caso B clasificado como Agente": evaluaciones["caso_B"]["control_flujo"] == "dinamico_llm",
    "Caso A clasificado como no-agente": evaluaciones["caso_A"]["control_flujo"] != "dinamico_llm",
    "Caso C clasificado como no-agente": evaluaciones["caso_C"]["control_flujo"] != "dinamico_llm",
    "Ciclo agéntico mapeado (4 componentes)": len(ciclo_agéntico_caso_B["componentes"]) == 4,
    "ARD contiene criterios de aceptación": len(ard["criterios_de_aceptacion"]) >= 3,
    "ARD exportado a Markdown": os.path.exists(nombre_archivo),
}

aprobadas = 0
for nombre_check, resultado in validaciones.items():
    estado = "✅" if resultado else "❌"
    print(f"  {estado} {nombre_check}")
    if resultado:
        aprobadas += 1

print(f"\n{'='*60}")
print(f"  Resultado: {aprobadas}/{len(validaciones)} verificaciones aprobadas")

if aprobadas == len(validaciones):
    print("  🎉 LAB COMPLETADO EXITOSAMENTE")
else:
    print(f"  ⚠️  Revisa los ítems marcados con ❌ antes de entregar.")
```

**Criterio de aprobación**: 9/9 verificaciones en verde (✅).

---

## 8. Resolución de Problemas

---

### Problema 1: Error de autenticación en la API de OpenAI

**Síntomas**:
```
openai.AuthenticationError: Error code: 401 - {'error': {'message': 'Incorrect API key provided...'}}
```
O bien: la celda de configuración muestra `❌ Error: API Key no encontrada`.

**Causa**: El archivo `.env` no existe, está en el directorio incorrecto, o la clave tiene un formato incorrecto (espacios adicionales, comillas incluidas en el valor).

**Solución**:

```bash
# 1. Verifica que el archivo .env existe en el directorio actual
ls -la .env          # macOS/Linux
dir .env             # Windows

# 2. Verifica el contenido (sin mostrar la clave completa)
python -c "
from dotenv import load_dotenv
import os
load_dotenv()
key = os.getenv('OPENAI_API_KEY', '')
print(f'Longitud de la clave: {len(key)} caracteres')
print(f'Prefijo correcto: {key.startswith(\"sk-\")}')
print(f'Sin espacios: {key == key.strip()}')
"

# 3. Si el archivo no existe o tiene errores, recréalo:
# Abre un editor de texto y crea el archivo .env con exactamente esta línea:
# OPENAI_API_KEY=sk-tu-clave-aqui
# Sin comillas, sin espacios antes o después del signo =
```

> Si el problema persiste, verifica en [platform.openai.com](https://platform.openai.com) que tu API key tiene crédito disponible y no ha expirado.

---

### Problema 2: El ARD exportado a Markdown tiene formato incorrecto o está vacío

**Síntomas**: El archivo `.md` se crea pero al abrirlo está vacío, tiene caracteres extraños o las tablas no se renderizan correctamente en la vista previa de Jupyter Lab.

**Causa**: El diccionario `ard` fue modificado accidentalmente (claves eliminadas o valores con caracteres especiales), o la función `exportar_ard_markdown` fue ejecutada antes de que el diccionario `ard` estuviera completamente definido.

**Solución**:

```python
# 1. Verifica que el diccionario ard tiene todas las secciones requeridas
secciones = ["metadata", "justificacion_arquitectura_agéntica",
             "componentes_agénticos", "nivel_sofisticacion",
             "criterios_de_aceptacion", "restricciones_y_riesgos"]

for s in secciones:
    print(f"{'✅' if s in ard else '❌'} {s}: {type(ard.get(s, 'FALTA')).__name__}")

# 2. Si el archivo existe pero está vacío, verifica su tamaño
import os
if os.path.exists(nombre_archivo):
    size = os.path.getsize(nombre_archivo)
    print(f"\nTamaño del archivo: {size} bytes")
    if size < 100:
        print("❌ El archivo está casi vacío. Re-ejecuta la celda de exportación.")
    else:
        print("✅ El archivo tiene contenido.")

# 3. Para re-generar el archivo, asegúrate de ejecutar las celdas en orden:
# Celda 1 (config) → Celda 2 (casos) → Celda 3 (rúbrica) → 
# Celda 4 (ciclo) → Celda 5 (demo API) → Celda 6 (ARD) → Celda 7 (export)
# Usa: Kernel → Restart Kernel and Run All Cells
```

> Si el problema persiste, usa **Kernel → Restart Kernel and Run All Cells** para ejecutar todas las celdas en orden desde cero.

---

## 9. Limpieza del Entorno

Una vez completado el lab, ejecuta los siguientes pasos:

```python
# En el notebook: celda de limpieza
# Eliminar variables sensibles de la memoria del kernel
del client
del api_key
print("✅ Variables sensibles eliminadas de la memoria del kernel.")
print("   Recuerda NO subir el archivo .env a ningún repositorio.")
```

```bash
# En la terminal: desactivar el entorno virtual
deactivate

# Opcional: si ya no necesitas el entorno virtual de este lab
# (NO eliminar si planeas retomar el lab)
# rm -rf venv_lab01    # macOS/Linux
# rmdir /s venv_lab01  # Windows
```

**Archivos generados en este lab** (conservar para referencia):

| Archivo | Descripción | ¿Conservar? |
|---|---|---|
| `lab_01_analisis_agentes.ipynb` | Notebook principal con todo el análisis | ✅ Sí |
| `ARD-001_analisis_financiero_YYYYMMDD.md` | Documento de requisitos agénticos | ✅ Sí (entregable) |
| `.env` | API Key (¡nunca compartir!) | ✅ Local únicamente |

---

## 10. Resumen

### Conceptos Aplicados en este Lab

En este laboratorio aplicaste la taxonomía conceptual de la Lección 1.1 a casos reales:

| Caso | Paradigma Correcto | Razón Principal |
|---|---|---|
| Soporte Técnico (A) | 🗨️ Chatbot avanzado (con RAG) | Alta tipificación; 85% de casos documentados; flujo predecible |
| Análisis Financiero (B) | 🤖 Agente de IA | Control dinámico; herramientas condicionales; adaptación ante hallazgos |
| Automatización RPA (C) | ⚙️ Flujo de IA | Proceso 100% predefinido; sin variabilidad; pipeline lineal suficiente |

### Lecciones Clave

1. **Más herramientas ≠ más agente**: el Caso C tiene 5 integraciones pero es un flujo, no un agente. Lo que define a un agente es la **autonomía decisional**, no la cantidad de herramientas.

2. **El nivel de sofisticación importa**: el Caso B se clasificó como **deliberativo** (no autónomo), porque opera dentro de un objetivo definido por el usuario. Los agentes completamente autónomos son raros en producción empresarial.

3. **Un ARD sin criterios medibles no es útil**: cada criterio de aceptación debe tener un umbral numérico y un método de verificación concreto para que sea accionable en un proyecto real.

4. **La elección de paradigma tiene consecuencias**: usar un agente donde un flujo es suficiente añade latencia, costo y complejidad de depuración innecesarios. La decisión correcta requiere análisis, no intuición.

### Próximos Pasos

En la **Lección 1.2** profundizarás en los cuatro componentes del ciclo agéntico que mapeaste en este lab (razonamiento, planificación, acción, observación) y aprenderás cómo interactúan técnicamente en implementaciones reales con LangGraph.

### Recursos Adicionales

| Recurso | Descripción |
|---|---|
| [ReAct: Synergizing Reasoning and Acting (Yao et al., 2022)](https://arxiv.org/abs/2210.03629) | Paper original del patrón que sustenta el razonamiento agéntico |
| [LangChain — Conceptos de Agentes](https://python.langchain.com/docs/concepts/agents/) | Documentación oficial sobre agentes y herramientas |
| [Lilian Weng — LLM-powered Autonomous Agents](https://lilianweng.github.io/posts/2023-06-23-agent/) | Referencia técnica comprensiva sobre arquitecturas agénticas |
| [OpenAI — Function Calling Guide](https://platform.openai.com/docs/guides/function-calling) | Base técnica del mecanismo de acción en agentes OpenAI |

---

> **Nota para instructores**: Este lab no requiere que los estudiantes implementen un agente funcional. Si un estudiante completa el lab antes de los 28 minutos, puede extenderlo modificando las evaluaciones de la rúbrica para el Caso A (¿podría justificarse un agente si el volumen sube a 10,000 consultas/día y se añaden integraciones con CRM y ERP?) y redactando un segundo ARD para ese escenario extendido.
