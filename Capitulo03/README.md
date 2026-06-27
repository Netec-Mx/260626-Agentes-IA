---LAB_START---
LAB_ID: 03-00-01
---MARKDOWN---
# Práctica 3 — Desplegar un Agente Simple en LangGraph

## 1. Metadatos

| Campo | Valor |
|---|---|
| **Duración estimada** | 46 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (*Apply*) |
| **Costo estimado de API** | ~$0.50 USD (GPT-4o-mini) |
| **Lab anterior requerido** | 01-00-01 y 02-00-01 |

---

## 2. Descripción General

En este laboratorio construirás un **agente de investigación y síntesis** funcional de extremo a extremo utilizando LangGraph 0.2.x. El agente implementará el ciclo completo **ReAct** (Razonar → Actuar → Observar) sobre un grafo de estado explícito, con acceso a tres herramientas personalizadas: búsqueda web simulada, calculadora matemática y consulta a base de conocimiento local en JSON. Configurarás LangSmith para capturar trazas completas de cada ejecución y, al finalizar, compararás la solución con una implementación equivalente en LlamaIndex para entender las diferencias arquitectónicas entre ambos frameworks.

---

## 3. Objetivos de Aprendizaje

- [ ] Implementar un agente funcional end-to-end con LangGraph usando al menos dos herramientas personalizadas y un ciclo ReAct completo.
- [ ] Configurar LangSmith para capturar y visualizar trazas de observabilidad del agente.
- [ ] Comparar el enfoque de LangGraph con LlamaIndex para el mismo caso de uso, identificando ventajas y limitaciones de cada framework.
- [ ] Implementar un router básico que seleccione la herramienta adecuada según el tipo de consulta.
- [ ] Manejar errores de herramientas dentro del grafo de estado sin interrumpir la ejecución del agente.

---

## 4. Prerrequisitos

### Conocimiento previo
- Haber completado **Lab 01-00-01** (conceptos de agentes de IA) y **Lab 02-00-01** (fundamentos de LangChain).
- Comprensión básica de `async/await` en Python.
- Familiaridad con `TypedDict` y tipos genéricos de Python.

### Acceso y cuentas requeridas
- **OpenAI API Key** con créditos disponibles (mínimo $1 USD recomendado).
- **Cuenta en LangSmith** (gratuita en [smith.langchain.com](https://smith.langchain.com)) — necesitas tu `LANGCHAIN_API_KEY`.
- **Docker Desktop** instalado y en ejecución (para validación del entorno).
- Conexión a internet estable (≥10 Mbps).

---

## 5. Entorno del Laboratorio

### Hardware mínimo requerido

| Componente | Mínimo | Recomendado |
|---|---|---|
| CPU | 4 núcleos (i5 8va gen / Ryzen 5) | 8 núcleos |
| RAM | 16 GB | 32 GB |
| Almacenamiento libre | 2 GB (para este lab) | 5 GB |
| GPU | No requerida | No requerida |

### Software requerido

| Paquete | Versión |
|---|---|
| Python | 3.11.x |
| LangGraph | 0.2.x |
| LangChain | 0.3.x |
| LangSmith SDK | 0.1.x |
| LlamaIndex | 0.11.x |
| OpenAI Python SDK | 1.40.x |
| Pydantic | 2.8.x |

### Configuración del entorno virtual

Abre una terminal y ejecuta los siguientes comandos. **Usa un entorno virtual separado** para este lab.

```bash
# 1. Crear directorio del lab
mkdir lab-03-00-01 && cd lab-03-00-01

# 2. Crear entorno virtual Python 3.11
python3.11 -m venv .venv

# 3. Activar el entorno virtual
# En Linux/macOS:
source .venv/bin/activate
# En Windows (PowerShell):
# .venv\Scripts\Activate.ps1

# 4. Actualizar pip
pip install --upgrade pip

# 5. Instalar dependencias del lab
pip install \
  "langgraph==0.2.*" \
  "langchain==0.3.*" \
  "langchain-openai==0.2.*" \
  "langsmith==0.1.*" \
  "llama-index==0.11.*" \
  "llama-index-llms-openai==0.3.*" \
  "openai==1.40.*" \
  "pydantic==2.8.*"

# 6. Verificar instalaciones
python -c "import langgraph; print('LangGraph:', langgraph.__version__)"
python -c "import langchain; print('LangChain:', langchain.__version__)"
python -c "import llama_index; print('LlamaIndex OK')"
```

### Configuración de variables de entorno

Crea el archivo `.env` en el directorio del lab:

```bash
# Crear archivo .env (NO subas este archivo a Git)
cat > .env << 'EOF'
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxx
LANGCHAIN_TRACING_V2=true
LANGCHAIN_ENDPOINT=https://api.smith.langchain.com
LANGCHAIN_API_KEY=ls__xxxxxxxxxxxxxxxxxxxxxxxxxxxx
LANGCHAIN_PROJECT=lab-03-agente-langgraph
EOF
```

> ⚠️ **Seguridad**: Reemplaza los valores `xx...` con tus claves reales. Nunca compartas ni subas el archivo `.env` a repositorios públicos.

---

## 6. Pasos del Laboratorio

---

### Paso 1 — Definir el esquema de estado y las herramientas del agente

**Objetivo**: Crear la estructura de datos del agente y las tres herramientas personalizadas (búsqueda simulada, calculadora y base de conocimiento local).

**Duración estimada**: 10 minutos

#### Instrucciones

1. Crea el archivo `tools.py` con el siguiente contenido:

```python
# tools.py
"""
Herramientas personalizadas para el agente de investigación y síntesis.
Todas las herramientas usan datos simulados (mock) para evitar dependencias externas.
"""
import json
import math
import re
from typing import Optional
from langchain_core.tools import tool

# ─────────────────────────────────────────────
# Base de conocimiento local (simulada en JSON)
# ─────────────────────────────────────────────
KNOWLEDGE_BASE = {
    "langgraph": {
        "descripcion": "Framework de orquestación de agentes basado en grafos de estado dirigidos.",
        "version_actual": "0.2.x",
        "casos_uso": ["agentes iterativos", "flujos con ciclos", "multi-agente"],
        "empresa": "LangChain Inc."
    },
    "langsmith": {
        "descripcion": "Plataforma de observabilidad, trazabilidad y evaluación para agentes LangChain.",
        "version_actual": "0.1.x",
        "casos_uso": ["depuración", "monitoreo en producción", "evaluación de regresión"],
        "empresa": "LangChain Inc."
    },
    "llamaindex": {
        "descripcion": "Framework especializado en ingesta, indexación y consulta de datos para IA.",
        "version_actual": "0.11.x",
        "casos_uso": ["RAG corporativo", "agentes de conocimiento", "múltiples fuentes de datos"],
        "empresa": "LlamaIndex Inc."
    },
    "crewai": {
        "descripcion": "Framework declarativo para sistemas multi-agente con roles y tareas definidas.",
        "version_actual": "0.70.x",
        "casos_uso": ["investigación + redacción", "flujos colaborativos", "agentes especializados"],
        "empresa": "CrewAI Inc."
    },
    "react": {
        "descripcion": "Patrón de razonamiento agéntico: Razonar, Actuar, Observar en ciclos iterativos.",
        "origen": "Paper de Yao et al., 2022",
        "ventaja": "Permite al agente corregir errores basándose en observaciones anteriores"
    }
}

# ─────────────────────────────────────────────
# Resultados de búsqueda simulados
# ─────────────────────────────────────────────
MOCK_SEARCH_RESULTS = {
    "langgraph": "LangGraph 0.2 introduce mejoras en la gestión de estado persistente y soporte nativo para streaming de nodos.",
    "langsmith": "LangSmith lanzó evaluación automática con LLM-as-judge en su versión más reciente, reduciendo el tiempo de evaluación un 60%.",
    "llamaindex": "LlamaIndex 0.11 incluye soporte para índices multi-modal y conectores para más de 100 fuentes de datos.",
    "agentes ia": "Los agentes de IA basados en LLMs han crecido un 340% en adopción empresarial durante 2024 según Gartner.",
    "default": "No se encontraron resultados específicos. Intenta con términos más precisos como 'langgraph', 'langsmith' o 'llamaindex'."
}


@tool
def buscar_web(consulta: str) -> str:
    """
    Busca información actualizada en la web sobre frameworks de IA y agentes.
    Úsala cuando necesites datos recientes o noticias sobre LangGraph, LangSmith, LlamaIndex u otros frameworks.
    
    Args:
        consulta: Término o pregunta de búsqueda (ej: 'novedades langgraph 2024')
    
    Returns:
        Resultado de búsqueda como texto plano.
    """
    consulta_lower = consulta.lower()
    for clave, resultado in MOCK_SEARCH_RESULTS.items():
        if clave in consulta_lower:
            return f"[Búsqueda web simulada] {resultado}"
    return f"[Búsqueda web simulada] {MOCK_SEARCH_RESULTS['default']}"


@tool
def calcular(expresion: str) -> str:
    """
    Evalúa expresiones matemáticas de forma segura.
    Úsala para cálculos numéricos como porcentajes, operaciones aritméticas o funciones matemáticas básicas.
    
    Args:
        expresion: Expresión matemática como string (ej: '2 + 2', '15 * 0.21', 'sqrt(144)')
    
    Returns:
        Resultado del cálculo como string.
    """
    # Sanitizar la expresión: solo permitir caracteres matemáticos seguros
    expresion_limpia = re.sub(r'[^0-9+\-*/().,\s]', '', expresion.replace('sqrt', ''))
    
    # Manejar sqrt explícitamente
    if 'sqrt' in expresion:
        numero = re.search(r'sqrt\((\d+\.?\d*)\)', expresion)
        if numero:
            resultado = math.sqrt(float(numero.group(1)))
            return f"sqrt({numero.group(1)}) = {resultado}"
    
    try:
        # Evaluar solo operaciones aritméticas básicas
        resultado = eval(expresion_limpia, {"__builtins__": {}}, {})
        return f"{expresion} = {resultado}"
    except Exception as e:
        return f"Error al calcular '{expresion}': {str(e)}. Verifica la expresión matemática."


@tool
def consultar_base_conocimiento(tema: str) -> str:
    """
    Consulta la base de conocimiento local sobre frameworks de IA agéntica.
    Úsala cuando necesites información estructurada y confiable sobre LangGraph, LangSmith, LlamaIndex, CrewAI o el patrón ReAct.
    
    Args:
        tema: Nombre del framework o concepto a consultar (ej: 'langgraph', 'react', 'crewai')
    
    Returns:
        Información estructurada del tema en formato JSON como string.
    """
    tema_lower = tema.lower().strip()
    
    # Búsqueda exacta
    if tema_lower in KNOWLEDGE_BASE:
        datos = KNOWLEDGE_BASE[tema_lower]
        return f"[Base de conocimiento] {json.dumps(datos, ensure_ascii=False, indent=2)}"
    
    # Búsqueda parcial
    for clave, datos in KNOWLEDGE_BASE.items():
        if tema_lower in clave or clave in tema_lower:
            return f"[Base de conocimiento] {json.dumps(datos, ensure_ascii=False, indent=2)}"
    
    temas_disponibles = list(KNOWLEDGE_BASE.keys())
    return f"[Base de conocimiento] Tema '{tema}' no encontrado. Temas disponibles: {temas_disponibles}"
```

2. Verifica que las herramientas funcionan de forma independiente:

```bash
python -c "
from tools import buscar_web, calcular, consultar_base_conocimiento
print(buscar_web.invoke({'consulta': 'novedades langgraph'}))
print(calcular.invoke({'expresion': '15 * 0.21'}))
print(consultar_base_conocimiento.invoke({'tema': 'langsmith'}))
"
```

**Salida esperada**:
```
[Búsqueda web simulada] LangGraph 0.2 introduce mejoras en la gestión de estado persistente...
15 * 0.21 = 3.15
[Base de conocimiento] {
  "descripcion": "Plataforma de observabilidad, trazabilidad y evaluación...
```

**Verificación**: Las tres herramientas deben responder sin errores. Si alguna falla, revisa que el archivo `tools.py` esté en el directorio correcto y que el entorno virtual esté activado.

---

### Paso 2 — Definir el grafo de estado y los nodos ReAct

**Objetivo**: Construir el `StateGraph` de LangGraph con el esquema de estado tipado y los nodos del ciclo ReAct: razonamiento, selección de herramienta y observación.

**Duración estimada**: 12 minutos

#### Instrucciones

1. Crea el archivo `agent_graph.py`:

```python
# agent_graph.py
"""
Agente de investigación y síntesis implementado con LangGraph.
Implementa el ciclo ReAct completo: Razonar → Actuar → Observar.
"""
import os
from typing import TypedDict, Annotated, List, Optional
import operator
from dotenv import load_dotenv

from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, ToolMessage, BaseMessage
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode

from tools import buscar_web, calcular, consultar_base_conocimiento

# Cargar variables de entorno desde .env
load_dotenv()

# ─────────────────────────────────────────────
# 1. Esquema de estado del agente
# ─────────────────────────────────────────────
class AgentState(TypedDict):
    """
    Estado completo del agente de investigación.
    Cada campo persiste y evoluciona a lo largo del ciclo ReAct.
    """
    messages: Annotated[List[BaseMessage], operator.add]  # historial de mensajes acumulado
    iteraciones: int                                        # contador de ciclos ReAct
    herramienta_usada: Optional[str]                       # última herramienta invocada
    error_ultimo: Optional[str]                            # último error capturado (si existe)


# ─────────────────────────────────────────────
# 2. Configurar el LLM con herramientas enlazadas
# ─────────────────────────────────────────────
HERRAMIENTAS = [buscar_web, calcular, consultar_base_conocimiento]

llm = ChatOpenAI(
    model="gpt-4o-mini",
    temperature=0,
    api_key=os.getenv("OPENAI_API_KEY")
)

# Vincular herramientas al LLM para que pueda invocarlas mediante tool_calls
llm_con_herramientas = llm.bind_tools(HERRAMIENTAS)


# ─────────────────────────────────────────────
# 3. Nodos del grafo
# ─────────────────────────────────────────────

def nodo_razonamiento(state: AgentState) -> dict:
    """
    Nodo de razonamiento: el LLM analiza el estado actual y decide
    si invocar una herramienta o generar la respuesta final.
    """
    # Construir prompt de sistema con contexto del agente
    system_prompt = """Eres un agente de investigación y síntesis especializado en frameworks de IA agéntica.
    
Tu misión es responder preguntas usando las herramientas disponibles:
- buscar_web: para información reciente o noticias
- calcular: para operaciones matemáticas
- consultar_base_conocimiento: para información estructurada sobre frameworks (LangGraph, LangSmith, LlamaIndex, CrewAI, ReAct)

Instrucciones:
1. Analiza la pregunta del usuario.
2. Si necesitas información, usa la herramienta más apropiada.
3. Después de obtener resultados, sintetiza una respuesta clara y concisa.
4. Si ya tienes suficiente información, responde directamente sin usar más herramientas.
5. Máximo 4 iteraciones de herramientas por consulta."""

    from langchain_core.messages import SystemMessage
    
    # Incluir mensaje de sistema solo en la primera iteración
    if state["iteraciones"] == 0:
        mensajes_con_sistema = [SystemMessage(content=system_prompt)] + state["messages"]
    else:
        mensajes_con_sistema = state["messages"]
    
    # Llamar al LLM
    respuesta = llm_con_herramientas.invoke(mensajes_con_sistema)
    
    return {
        "messages": [respuesta],
        "iteraciones": state["iteraciones"] + 1
    }


def nodo_manejo_errores(state: AgentState) -> dict:
    """
    Nodo de manejo de errores: captura errores de herramientas y
    añade un mensaje de recuperación para que el agente continúe.
    """
    mensaje_recuperacion = AIMessage(
        content=f"Se produjo un error en la herramienta. Intentaré responder con la información disponible hasta ahora."
    )
    return {
        "messages": [mensaje_recuperacion],
        "error_ultimo": "error_capturado"
    }


# ─────────────────────────────────────────────
# 4. Router: decide el siguiente nodo
# ─────────────────────────────────────────────

def router(state: AgentState) -> str:
    """
    Función de enrutamiento condicional.
    Evalúa el último mensaje del agente y decide:
    - "herramientas": si el LLM solicitó invocar una herramienta
    - END: si el LLM generó una respuesta final
    - END: si se alcanzó el límite de iteraciones (protección contra bucles infinitos)
    """
    # Protección contra bucles infinitos
    if state["iteraciones"] >= 5:
        return END
    
    ultimo_mensaje = state["messages"][-1]
    
    # Verificar si el LLM realizó una llamada a herramienta (tool_calls)
    if hasattr(ultimo_mensaje, "tool_calls") and ultimo_mensaje.tool_calls:
        # Registrar qué herramienta se va a usar
        nombre_herramienta = ultimo_mensaje.tool_calls[0]["name"]
        return "herramientas"
    
    # Sin tool_calls: el LLM generó respuesta final
    return END


# ─────────────────────────────────────────────
# 5. Construcción del grafo
# ─────────────────────────────────────────────

def construir_grafo() -> StateGraph:
    """
    Construye y compila el grafo de estado del agente ReAct.
    
    Flujo del grafo:
    START → razonamiento → [router] → herramientas → razonamiento (ciclo)
                                    ↓
                                   END
    """
    # Crear nodo de herramientas usando ToolNode de LangGraph
    # ToolNode ejecuta automáticamente la herramienta indicada en tool_calls
    nodo_herramientas = ToolNode(HERRAMIENTAS)
    
    # Inicializar el grafo con el esquema de estado
    grafo = StateGraph(AgentState)
    
    # Agregar nodos
    grafo.add_node("razonamiento", nodo_razonamiento)
    grafo.add_node("herramientas", nodo_herramientas)
    
    # Definir punto de entrada
    grafo.set_entry_point("razonamiento")
    
    # Arista condicional: después del razonamiento, el router decide
    grafo.add_conditional_edges(
        "razonamiento",  # nodo origen
        router,          # función de decisión
        {
            "herramientas": "herramientas",  # si router retorna "herramientas"
            END: END                          # si router retorna END
        }
    )
    
    # Arista fija: después de ejecutar herramientas, siempre volver a razonar
    grafo.add_edge("herramientas", "razonamiento")
    
    # Compilar el grafo
    return grafo.compile()


# Instancia global del agente compilado
agente = construir_grafo()
```

2. Verifica que el grafo se construye sin errores:

```bash
python -c "
from agent_graph import agente
print('Grafo compilado exitosamente')
print('Nodos:', list(agente.get_graph().nodes.keys()))
"
```

**Salida esperada**:
```
Grafo compilado exitosamente
Nodos: ['__start__', 'razonamiento', 'herramientas', '__end__']
```

**Verificación**: Deben aparecer exactamente los nodos `razonamiento` y `herramientas` además de los nodos internos `__start__` y `__end__`. Si ves un `ImportError`, verifica que `tools.py` esté en el mismo directorio.

---

### Paso 3 — Configurar LangSmith y ejecutar el agente

**Objetivo**: Activar la observabilidad con LangSmith y ejecutar el agente con tres consultas de prueba que ejerciten las tres herramientas.

**Duración estimada**: 10 minutos

#### Instrucciones

1. Crea el archivo `run_agent.py`:

```python
# run_agent.py
"""
Script de ejecución del agente con observabilidad LangSmith activada.
Las variables de entorno en .env activan automáticamente el tracing.
"""
import os
from dotenv import load_dotenv
from langchain_core.messages import HumanMessage

# Cargar .env ANTES de importar el agente (activa LangSmith tracing)
load_dotenv()

# Verificar configuración de LangSmith
langsmith_activo = os.getenv("LANGCHAIN_TRACING_V2", "false").lower() == "true"
proyecto = os.getenv("LANGCHAIN_PROJECT", "default")
print(f"✅ LangSmith tracing: {'ACTIVO' if langsmith_activo else 'INACTIVO'}")
print(f"📁 Proyecto LangSmith: {proyecto}")
print("-" * 60)

from agent_graph import agente


def ejecutar_consulta(pregunta: str, etiqueta: str = "") -> str:
    """
    Ejecuta una consulta en el agente y retorna la respuesta final.
    
    Args:
        pregunta: Pregunta del usuario
        etiqueta: Etiqueta descriptiva para identificar la prueba en logs
    
    Returns:
        Respuesta final del agente como string
    """
    print(f"\n{'='*60}")
    print(f"🔍 Consulta {etiqueta}: {pregunta}")
    print('='*60)
    
    # Estado inicial del agente
    estado_inicial = {
        "messages": [HumanMessage(content=pregunta)],
        "iteraciones": 0,
        "herramienta_usada": None,
        "error_ultimo": None
    }
    
    # Ejecutar el grafo con streaming de eventos para visualizar el flujo
    respuesta_final = ""
    
    for evento in agente.stream(estado_inicial, stream_mode="values"):
        # Obtener el último mensaje del estado actual
        if "messages" in evento and evento["messages"]:
            ultimo = evento["messages"][-1]
            
            # Mostrar actividad del agente en tiempo real
            tipo = type(ultimo).__name__
            if tipo == "AIMessage":
                if hasattr(ultimo, "tool_calls") and ultimo.tool_calls:
                    herramienta = ultimo.tool_calls[0]["name"]
                    args = ultimo.tool_calls[0]["args"]
                    print(f"  🔧 Invocando herramienta: {herramienta}({args})")
                else:
                    # Respuesta final del agente
                    respuesta_final = ultimo.content
                    print(f"  💬 Respuesta: {respuesta_final[:200]}...")
            elif tipo == "ToolMessage":
                print(f"  📥 Resultado herramienta: {ultimo.content[:150]}...")
    
    print(f"\n✅ Iteraciones usadas: {evento.get('iteraciones', '?')}")
    return respuesta_final


if __name__ == "__main__":
    # ─── Consulta 1: Usa la base de conocimiento ───
    ejecutar_consulta(
        pregunta="¿Qué es LangGraph y para qué casos de uso es más adecuado?",
        etiqueta="[1/3] Base de conocimiento"
    )
    
    # ─── Consulta 2: Usa búsqueda web ───
    ejecutar_consulta(
        pregunta="¿Cuáles son las novedades más recientes de LlamaIndex?",
        etiqueta="[2/3] Búsqueda web"
    )
    
    # ─── Consulta 3: Usa calculadora + base de conocimiento ───
    ejecutar_consulta(
        pregunta="Si un agente de IA procesa 150 solicitudes por hora y cada una cuesta $0.003, ¿cuánto costaría operar el agente durante 8 horas? Además, ¿qué framework usarías para implementarlo?",
        etiqueta="[3/3] Calculadora + base de conocimiento"
    )
    
    print("\n" + "="*60)
    print("🎉 Lab completado. Revisa las trazas en LangSmith:")
    print(f"   https://smith.langchain.com → Proyecto: {proyecto}")
    print("="*60)
```

2. Ejecuta el agente:

```bash
python run_agent.py
```

**Salida esperada** (fragmento representativo):
```
✅ LangSmith tracing: ACTIVO
📁 Proyecto LangSmith: lab-03-agente-langgraph
------------------------------------------------------------

============================================================
🔍 Consulta [1/3] Base de conocimiento: ¿Qué es LangGraph...
============================================================
  🔧 Invocando herramienta: consultar_base_conocimiento({'tema': 'langgraph'})
  📥 Resultado herramienta: [Base de conocimiento] {
  "descripcion": "Framework de orquestación de agentes...
  💬 Respuesta: LangGraph es un framework de orquestación de agentes...

✅ Iteraciones usadas: 2
```

3. Abre [smith.langchain.com](https://smith.langchain.com), navega al proyecto `lab-03-agente-langgraph` y verifica que aparecen las trazas de las tres ejecuciones.

**Verificación**: En LangSmith debes ver tres trazas, cada una con subniveles que muestran las llamadas al LLM y las invocaciones de herramientas. Los nodos exitosos aparecen en verde.

---

### Paso 4 — Implementar el router avanzado con clasificación de consultas

**Objetivo**: Añadir un nodo de clasificación previo que enrute la consulta al tipo de herramienta más apropiado antes de invocar el LLM, implementando el patrón **router-specialists**.

**Duración estimada**: 8 minutos

#### Instrucciones

1. Crea el archivo `router_classifier.py`:

```python
# router_classifier.py
"""
Router de clasificación de consultas.
Implementa el patrón router-specialists: clasifica la intención
de la consulta y sugiere la herramienta primaria al agente.
"""
import re
from typing import Literal


# Patrones de clasificación por tipo de consulta
PATRONES_CALCULO = [
    r'\d+\s*[\+\-\*\/]\s*\d+',  # expresiones aritméticas
    r'cuánto\s+cuesta',
    r'calcula',
    r'porcentaje',
    r'precio',
    r'costo',
    r'total',
    r'suma',
    r'multiplica'
]

PATRONES_BUSQUEDA = [
    r'novedades',
    r'recientes',
    r'últimas noticias',
    r'actualización',
    r'nuevo en',
    r'lanzamiento',
    r'anuncio'
]

PATRONES_CONOCIMIENTO = [
    r'qué es',
    r'cómo funciona',
    r'explica',
    r'describe',
    r'características',
    r'diferencia entre',
    r'cuándo usar',
    r'casos de uso',
    r'langgraph|langsmith|llamaindex|crewai|react'
]


def clasificar_consulta(
    consulta: str
) -> Literal["calculo", "busqueda_web", "base_conocimiento", "mixta"]:
    """
    Clasifica la intención principal de una consulta.
    
    Args:
        consulta: Texto de la consulta del usuario
    
    Returns:
        Categoría de la consulta: 'calculo', 'busqueda_web', 'base_conocimiento' o 'mixta'
    """
    consulta_lower = consulta.lower()
    
    puntuacion = {
        "calculo": 0,
        "busqueda_web": 0,
        "base_conocimiento": 0
    }
    
    for patron in PATRONES_CALCULO:
        if re.search(patron, consulta_lower):
            puntuacion["calculo"] += 1
    
    for patron in PATRONES_BUSQUEDA:
        if re.search(patron, consulta_lower):
            puntuacion["busqueda_web"] += 1
    
    for patron in PATRONES_CONOCIMIENTO:
        if re.search(patron, consulta_lower):
            puntuacion["base_conocimiento"] += 1
    
    # Determinar categoría dominante
    max_puntuacion = max(puntuacion.values())
    
    if max_puntuacion == 0:
        return "base_conocimiento"  # default
    
    categorias_dominantes = [k for k, v in puntuacion.items() if v == max_puntuacion]
    
    if len(categorias_dominantes) > 1:
        return "mixta"
    
    return categorias_dominantes[0]


def obtener_hint_herramienta(categoria: str) -> str:
    """
    Genera un hint de herramienta para incluir en el prompt del agente.
    
    Args:
        categoria: Categoría clasificada de la consulta
    
    Returns:
        Instrucción de herramienta sugerida para el prompt
    """
    hints = {
        "calculo": "HINT: Esta consulta requiere cálculo matemático. Usa la herramienta 'calcular' primero.",
        "busqueda_web": "HINT: Esta consulta requiere información reciente. Usa 'buscar_web' como primera herramienta.",
        "base_conocimiento": "HINT: Esta consulta es sobre frameworks de IA. Usa 'consultar_base_conocimiento' primero.",
        "mixta": "HINT: Esta consulta requiere múltiples herramientas. Planifica el orden de invocación."
    }
    return hints.get(categoria, "")


# ─── Demo del clasificador ───
if __name__ == "__main__":
    consultas_prueba = [
        "¿Cuánto cuesta procesar 1000 tokens?",
        "¿Cuáles son las novedades de LangSmith?",
        "¿Qué es el patrón ReAct?",
        "Si proceso 500 solicitudes y cada una cuesta $0.002, ¿cuánto gasto? ¿Y qué framework uso?"
    ]
    
    print("🔀 Demostración del Router Clasificador\n")
    for consulta in consultas_prueba:
        categoria = clasificar_consulta(consulta)
        hint = obtener_hint_herramienta(categoria)
        print(f"Consulta: {consulta[:60]}...")
        print(f"  → Categoría: {categoria}")
        print(f"  → {hint}\n")
```

2. Ejecuta la demo del clasificador:

```bash
python router_classifier.py
```

**Salida esperada**:
```
🔀 Demostración del Router Clasificador

Consulta: ¿Cuánto cuesta procesar 1000 tokens?...
  → Categoría: calculo
  → HINT: Esta consulta requiere cálculo matemático. Usa la herramienta 'calcular' primero.

Consulta: ¿Cuáles son las novedades de LangSmith?...
  → Categoría: busqueda_web
  → HINT: Esta consulta requiere información reciente. Usa 'buscar_web' como primera herramienta.
...
```

**Verificación**: Las cuatro consultas deben clasificarse correctamente. La consulta mixta debe retornar `mixta`.

---

### Paso 5 — Comparativa: implementar el mismo caso de uso con LlamaIndex

**Objetivo**: Implementar una versión equivalente del agente de investigación usando LlamaIndex ReActAgent para comparar arquitecturas, verbosidad del código y comportamiento.

**Duración estimada**: 6 minutos

#### Instrucciones

1. Crea el archivo `agent_llamaindex.py`:

```python
# agent_llamaindex.py
"""
Implementación del agente de investigación usando LlamaIndex.
Propósito: comparar con la implementación LangGraph del mismo caso de uso.
"""
import os
import json
from dotenv import load_dotenv
load_dotenv()

from llama_index.core.tools import FunctionTool
from llama_index.core.agent import ReActAgent
from llama_index.llms.openai import OpenAI

# ─── Base de conocimiento (misma que en LangGraph) ───
KNOWLEDGE_BASE = {
    "langgraph": {
        "descripcion": "Framework de orquestación de agentes basado en grafos de estado dirigidos.",
        "version_actual": "0.2.x",
        "casos_uso": ["agentes iterativos", "flujos con ciclos", "multi-agente"]
    },
    "langsmith": {
        "descripcion": "Plataforma de observabilidad, trazabilidad y evaluación para agentes LangChain.",
        "casos_uso": ["depuración", "monitoreo en producción", "evaluación de regresión"]
    }
}


# ─── Herramientas como funciones Python estándar ───
def buscar_web_li(consulta: str) -> str:
    """Busca información reciente sobre frameworks de IA."""
    if "llamaindex" in consulta.lower():
        return "LlamaIndex 0.11 incluye soporte para índices multi-modal y 100+ conectores."
    if "langgraph" in consulta.lower():
        return "LangGraph 0.2 mejora la gestión de estado persistente con streaming nativo."
    return "No se encontraron resultados específicos."


def consultar_kb_li(tema: str) -> str:
    """Consulta la base de conocimiento sobre frameworks de IA."""
    tema_lower = tema.lower()
    if tema_lower in KNOWLEDGE_BASE:
        return json.dumps(KNOWLEDGE_BASE[tema_lower], ensure_ascii=False)
    return f"Tema '{tema}' no encontrado. Disponibles: {list(KNOWLEDGE_BASE.keys())}"


def calcular_li(expresion: str) -> str:
    """Evalúa expresiones matemáticas básicas."""
    try:
        import re
        expr_limpia = re.sub(r'[^0-9+\-*/().,\s]', '', expresion)
        resultado = eval(expr_limpia, {"__builtins__": {}}, {})
        return f"{expresion} = {resultado}"
    except Exception as e:
        return f"Error: {str(e)}"


# ─── Construcción del agente LlamaIndex ───
herramientas_li = [
    FunctionTool.from_defaults(fn=buscar_web_li, name="buscar_web",
        description="Busca información reciente sobre frameworks de IA"),
    FunctionTool.from_defaults(fn=consultar_kb_li, name="consultar_base_conocimiento",
        description="Consulta base de conocimiento estructurada sobre LangGraph, LangSmith, etc."),
    FunctionTool.from_defaults(fn=calcular_li, name="calcular",
        description="Evalúa expresiones matemáticas")
]

llm_li = OpenAI(model="gpt-4o-mini", temperature=0)

agente_li = ReActAgent.from_tools(
    herramientas_li,
    llm=llm_li,
    verbose=True,
    max_iterations=5
)


# ─── Ejecutar consulta de comparación ───
if __name__ == "__main__":
    print("=" * 60)
    print("🦙 Agente LlamaIndex ReActAgent — Consulta de comparación")
    print("=" * 60)
    
    pregunta = "¿Qué es LangGraph y cuáles son sus casos de uso principales?"
    print(f"\nPregunta: {pregunta}\n")
    
    respuesta = agente_li.chat(pregunta)
    print(f"\n✅ Respuesta final:\n{respuesta}")
    
    print("\n" + "=" * 60)
    print("📊 TABLA COMPARATIVA: LangGraph vs LlamaIndex")
    print("=" * 60)
    
    tabla = """
    | Aspecto                    | LangGraph                        | LlamaIndex ReActAgent        |
    |----------------------------|----------------------------------|------------------------------|
    | Control del flujo          | Explícito (grafo de nodos)       | Implícito (loop interno)     |
    | Estado del agente          | TypedDict explícito y tipado     | Manejado internamente        |
    | Ciclos y bifurcaciones     | Nativo con aristas condicionales | Limitado al loop ReAct       |
    | Observabilidad             | LangSmith nativo                 | Callbacks manuales           |
    | Curva de aprendizaje       | Media-Alta                       | Baja-Media                   |
    | Caso ideal                 | Flujos complejos con estado      | RAG y consulta de documentos |
    | Líneas de código (este lab)| ~120 líneas                      | ~50 líneas                   |
    """
    print(tabla)
```

2. Ejecuta la comparativa:

```bash
python agent_llamaindex.py
```

**Salida esperada** (fragmento):
```
============================================================
🦙 Agente LlamaIndex ReActAgent — Consulta de comparación
============================================================
Thought: I need to look up information about LangGraph...
Action: consultar_base_conocimiento
Action Input: {"tema": "langgraph"}
Observation: {"descripcion": "Framework de orquestación...
...
✅ Respuesta final: LangGraph es un framework...
```

**Verificación**: El agente LlamaIndex debe responder la pregunta usando la herramienta `consultar_base_conocimiento`. La tabla comparativa debe imprimirse al final.

---

## 7. Validación y Pruebas

Ejecuta el siguiente script de validación completo para confirmar que todos los componentes del lab funcionan correctamente:

```python
# validate_lab.py
"""
Script de validación del Lab 03-00-01.
Ejecuta pruebas automáticas sobre todos los componentes implementados.
"""
import sys
import os
from dotenv import load_dotenv
load_dotenv()

resultados = []

def test(nombre: str, fn):
    """Ejecuta una prueba y registra el resultado."""
    try:
        fn()
        print(f"  ✅ PASS: {nombre}")
        resultados.append(("PASS", nombre))
    except Exception as e:
        print(f"  ❌ FAIL: {nombre} → {str(e)[:100]}")
        resultados.append(("FAIL", nombre, str(e)))


print("\n🧪 Validando Lab 03-00-01...\n")

# Test 1: Herramientas individuales
def test_herramientas():
    from tools import buscar_web, calcular, consultar_base_conocimiento
    r1 = buscar_web.invoke({"consulta": "langgraph"})
    assert "LangGraph" in r1, f"buscar_web no retornó resultado esperado: {r1}"
    r2 = calcular.invoke({"expresion": "10 * 5"})
    assert "50" in r2, f"calcular no retornó 50: {r2}"
    r3 = consultar_base_conocimiento.invoke({"tema": "langsmith"})
    assert "observabilidad" in r3.lower() or "trazabilidad" in r3.lower(), f"KB no retornó info de LangSmith: {r3}"

test("Herramientas personalizadas (buscar_web, calcular, consultar_base_conocimiento)", test_herramientas)


# Test 2: Construcción del grafo
def test_grafo():
    from agent_graph import agente
    nodos = list(agente.get_graph().nodes.keys())
    assert "razonamiento" in nodos, f"Nodo 'razonamiento' no encontrado: {nodos}"
    assert "herramientas" in nodos, f"Nodo 'herramientas' no encontrado: {nodos}"

test("Grafo LangGraph compilado con nodos correctos", test_grafo)


# Test 3: Router clasificador
def test_router():
    from router_classifier import clasificar_consulta
    assert clasificar_consulta("calcula 5 * 10") == "calculo"
    assert clasificar_consulta("novedades de langsmith") == "busqueda_web"
    assert clasificar_consulta("qué es el patrón ReAct") == "base_conocimiento"

test("Router clasificador de consultas", test_router)


# Test 4: Variables de entorno LangSmith
def test_langsmith_config():
    assert os.getenv("LANGCHAIN_TRACING_V2") == "true", "LANGCHAIN_TRACING_V2 no configurado"
    assert os.getenv("LANGCHAIN_API_KEY"), "LANGCHAIN_API_KEY no configurado"
    assert os.getenv("LANGCHAIN_PROJECT"), "LANGCHAIN_PROJECT no configurado"

test("Configuración LangSmith (variables de entorno)", test_langsmith_config)


# Test 5: Ejecución end-to-end del agente (consulta simple)
def test_ejecucion_agente():
    from agent_graph import agente
    from langchain_core.messages import HumanMessage
    estado = {
        "messages": [HumanMessage(content="¿Qué es LangSmith? Responde brevemente.")],
        "iteraciones": 0,
        "herramienta_usada": None,
        "error_ultimo": None
    }
    resultado = agente.invoke(estado)
    assert len(resultado["messages"]) > 1, "El agente no generó mensajes adicionales"
    ultima_respuesta = resultado["messages"][-1].content
    assert len(ultima_respuesta) > 20, f"Respuesta demasiado corta: {ultima_respuesta}"

test("Ejecución end-to-end del agente LangGraph", test_ejecucion_agente)


# Resumen
print(f"\n{'='*50}")
total = len(resultados)
pasados = sum(1 for r in resultados if r[0] == "PASS")
print(f"📊 Resultado: {pasados}/{total} pruebas pasadas")

if pasados == total:
    print("🎉 ¡Todos los tests pasaron! Lab completado exitosamente.")
else:
    fallidos = [r for r in resultados if r[0] == "FAIL"]
    print(f"⚠️  {len(fallidos)} test(s) fallaron. Revisa los errores arriba.")
    sys.exit(1)
```

Ejecuta la validación:

```bash
python validate_lab.py
```

**Salida esperada**:
```
🧪 Validando Lab 03-00-01...

  ✅ PASS: Herramientas personalizadas (buscar_web, calcular, consultar_base_conocimiento)
  ✅ PASS: Grafo LangGraph compilado con nodos correctos
  ✅ PASS: Router clasificador de consultas
  ✅ PASS: Configuración LangSmith (variables de entorno)
  ✅ PASS: Ejecución end-to-end del agente LangGraph

==================================================
📊 Resultado: 5/5 pruebas pasadas
🎉 ¡Todos los tests pasaron! Lab completado exitosamente.
```

### Verificación en LangSmith

1. Abre [smith.langchain.com](https://smith.langchain.com) en tu navegador.
2. Navega a **Projects** → `lab-03-agente-langgraph`.
3. Verifica que existen al menos **3 trazas** de ejecución del agente.
4. Haz clic en la traza de la Consulta 3 (mixta) y confirma que el árbol de ejecución muestra:
   - Nodo `razonamiento` → llamada al LLM con `tool_calls`
   - Nodo `herramientas` → ejecución de `calcular`
   - Nodo `razonamiento` → segunda llamada al LLM
   - Nodo `herramientas` → ejecución de `consultar_base_conocimiento`
   - Nodo `razonamiento` → respuesta final sin `tool_calls`

---

## 8. Resolución de Problemas

### Problema 1: `AuthenticationError` al ejecutar el agente

**Síntoma**: Al ejecutar `run_agent.py`, aparece el error:
```
openai.AuthenticationError: Error code: 401 - {'error': {'message': 'Incorrect API key provided'}}
```

**Causa**: La variable de entorno `OPENAI_API_KEY` no está configurada correctamente, o el archivo `.env` no se está cargando antes de la inicialización del cliente OpenAI.

**Solución**:
1. Verifica el contenido del archivo `.env`:
   ```bash
   cat .env | grep OPENAI_API_KEY
   # Debe mostrar: OPENAI_API_KEY=sk-...
   ```
2. Confirma que `load_dotenv()` se llama al inicio del script, **antes** de cualquier `import` que use la API key:
   ```python
   # CORRECTO: load_dotenv() primero
   from dotenv import load_dotenv
   load_dotenv()
   from agent_graph import agente  # importar después
   ```
3. Si el problema persiste, exporta la variable directamente en la terminal:
   ```bash
   export OPENAI_API_KEY="sk-tu-clave-real"
   python run_agent.py
   ```
4. Verifica que la API key tenga créditos disponibles en [platform.openai.com/usage](https://platform.openai.com/usage).

---

### Problema 2: Las trazas no aparecen en LangSmith

**Síntoma**: El agente ejecuta correctamente y produce respuestas, pero el proyecto `lab-03-agente-langgraph` no muestra trazas en la interfaz de LangSmith.

**Causa**: Una de las tres variables de entorno de LangSmith está mal configurada o ausente (`LANGCHAIN_TRACING_V2`, `LANGCHAIN_API_KEY`, o `LANGCHAIN_ENDPOINT`), o la `LANGCHAIN_API_KEY` de LangSmith (diferente a la de OpenAI) no es válida.

**Solución**:
1. Ejecuta el siguiente diagnóstico:
   ```bash
   python -c "
   from dotenv import load_dotenv
   import os
   load_dotenv()
   print('TRACING_V2:', os.getenv('LANGCHAIN_TRACING_V2'))
   print('ENDPOINT:', os.getenv('LANGCHAIN_ENDPOINT'))
   print('API_KEY set:', bool(os.getenv('LANGCHAIN_API_KEY')))
   print('PROJECT:', os.getenv('LANGCHAIN_PROJECT'))
   "
   ```
2. Verifica que `LANGCHAIN_TRACING_V2` sea exactamente `true` (minúsculas).
3. Obtén tu `LANGCHAIN_API_KEY` de LangSmith desde: **smith.langchain.com → Settings → API Keys**.
4. Confirma que el endpoint sea exactamente `https://api.smith.langchain.com` (sin barra final).
5. Si usas una cuenta nueva de LangSmith, espera 1-2 minutos para que las trazas aparezcan en la interfaz (puede haber latencia).

---

## 9. Limpieza del Entorno

Ejecuta los siguientes comandos para liberar recursos al finalizar el lab:

```bash
# 1. Desactivar el entorno virtual
deactivate

# 2. (Opcional) Eliminar el entorno virtual para liberar espacio
# ADVERTENCIA: Esto elimina todas las dependencias instaladas.
# Omite este paso si planeas retomar el lab.
rm -rf .venv

# 3. (Opcional) Archivar los archivos del lab
cd ..
zip -r lab-03-00-01-backup.zip lab-03-00-01/

# 4. Verificar que no quedan procesos de Python del lab activos
# En Linux/macOS:
ps aux | grep python | grep -v grep

# 5. (Opcional) Limpiar trazas de LangSmith del proyecto de prueba
# Desde la interfaz web: Projects → lab-03-agente-langgraph → Delete project
# O usando el SDK:
python -c "
from dotenv import load_dotenv; load_dotenv()
from langsmith import Client
c = Client()
# Listar runs del proyecto (no elimina, solo verifica)
runs = list(c.list_runs(project_name='lab-03-agente-langgraph', limit=5))
print(f'Trazas en proyecto: {len(runs)} (mostrando últimas 5)')
"
```

> 💡 **Nota para el siguiente lab**: Los archivos `tools.py` y `agent_graph.py` de este lab serán extendidos en el **Lab 04-00-01**. Conserva el directorio `lab-03-00-01/` como referencia.

---

## 10. Resumen

En este laboratorio implementaste un agente de investigación y síntesis completo usando LangGraph 0.2.x. Los conceptos clave aplicados fueron:

| Concepto | Implementación en el lab |
|---|---|
| **StateGraph con TypedDict** | `AgentState` con campos `messages`, `iteraciones`, `herramienta_usada`, `error_ultimo` |
| **Ciclo ReAct** | Nodos `razonamiento` → `herramientas` → `razonamiento` con arista condicional |
| **ToolNode** | Ejecución automática de herramientas a partir de `tool_calls` del LLM |
| **Router condicional** | Función `router()` que evalúa `tool_calls` y decide entre continuar o finalizar |
| **Observabilidad LangSmith** | Variables de entorno `LANGCHAIN_TRACING_V2` + `LANGCHAIN_API_KEY` |
| **Comparativa de frameworks** | LangGraph (control explícito del flujo) vs LlamaIndex (loop ReAct implícito) |

### Diferencias clave: LangGraph vs LlamaIndex

La comparativa del Paso 5 ilustró que **LangGraph** es la elección correcta cuando necesitas control granular sobre el flujo del agente, persistencia de estado entre nodos y capacidad de implementar bifurcaciones complejas. **LlamaIndex ReActAgent** ofrece una implementación más concisa ideal para casos donde el agente principal accede a fuentes de datos indexadas y el flujo es predecible.

### Próximos pasos

- **Lab 04-00-01**: Extenderás este agente añadiendo memoria persistente con checkpointing de LangGraph.
- **Lab 05-00-01**: Implementarás el patrón planner-executor sobre el grafo construido en este lab.

### Recursos adicionales

- [Documentación oficial de LangGraph](https://langchain-ai.github.io/langgraph/)
- [Guía de inicio de LangSmith](https://docs.smith.langchain.com/)
- [Paper original ReAct: Synergizing Reasoning and Acting](https://arxiv.org/abs/2210.03629)
- [LlamaIndex Agents Documentation](https://docs.llamaindex.ai/en/stable/module_guides/deploying/agents/)

---
LAB_END---
