---LAB_START---
LAB_ID: 05-00-01
---MARKDOWN---
# Práctica 5 — Construir una Tool MCP y Conectarla al Agente

## Metadatos

| Campo            | Valor                                                   |
|------------------|---------------------------------------------------------|
| **Duración**     | 46 minutos                                              |
| **Complejidad**  | Alta                                                    |
| **Nivel Bloom**  | Aplicar (Apply)                                         |
| **Lab anterior** | 04-00-01 (Agente LangGraph con herramientas locales)    |
| **Costo API est.**| ~$0.50 USD (GPT-4o-mini)                               |

---

## Descripción General

En este lab construirás un **servidor MCP completo** usando FastAPI que expone dos tools de negocio con contrato OpenAPI documentado: una herramienta de consulta a una base de datos de productos (SQLite) y una herramienta de envío de notificaciones (mock). El servidor implementa autenticación JWT con scopes diferenciados por tool. Luego reconfiguras el agente LangGraph del Lab 4 para consumir estas tools desde el servidor MCP en lugar de herramientas locales, aplicando los principios de separación de responsabilidades que define el protocolo MCP.

---

## Objetivos de Aprendizaje

- [ ] Implementar un servidor MCP con FastAPI que exponga al menos dos tools como endpoints REST con contrato OpenAPI documentado
- [ ] Diseñar esquemas de request/response con Pydantic 2.x incluyendo validación estricta y manejo de errores estandarizado con códigos HTTP
- [ ] Configurar autenticación JWT y autorización basada en scopes para controlar el acceso a cada tool del servidor MCP
- [ ] Conectar el agente LangGraph al servidor MCP y verificar el flujo completo de invocación con controles de seguridad activos
- [ ] Ejecutar pruebas de seguridad: acceso sin token, token expirado y scopes insuficientes

---

## Prerrequisitos

### Conocimiento previo
- Haber completado Labs **03-00-01** y **04-00-01** (agente LangGraph funcional)
- Comprensión de APIs REST y HTTP (métodos, códigos de estado, headers)
- Conocimiento básico de JWT (estructura, claims, expiración)
- Familiaridad con Pydantic v2 y validación de esquemas

### Acceso y credenciales
- API key de OpenAI configurada como variable de entorno `OPENAI_API_KEY`
- Docker Desktop instalado y en ejecución (para Redis, opcional)
- Puerto **8000** disponible en localhost

---

## Entorno de Laboratorio

### Hardware mínimo recomendado

| Recurso     | Mínimo            | Recomendado       |
|-------------|-------------------|-------------------|
| CPU         | 4 núcleos         | 8 núcleos         |
| RAM         | 16 GB             | 32 GB             |
| Disco libre | 20 GB SSD         | 20 GB SSD         |
| Red         | 10 Mbps           | 25 Mbps           |

### Software requerido

| Paquete               | Versión   | Rol en el lab                          |
|-----------------------|-----------|----------------------------------------|
| Python                | 3.11.x    | Runtime principal                      |
| FastAPI               | 0.115.x   | Servidor MCP                           |
| Uvicorn               | 0.30.x    | Servidor ASGI                          |
| Pydantic              | 2.8.x     | Esquemas y validación                  |
| python-jose[cryptography] | 3.3.x | Generación y validación de JWT         |
| LangGraph             | 0.2.x     | Agente consumidor de MCP               |
| LangChain             | 0.3.x     | Herramientas y cadenas                 |
| httpx                 | 0.27.x    | Cliente HTTP para integración y pruebas|
| SQLite3               | (stdlib)  | Base de datos de productos             |
| openai                | 1.40.x    | SDK de OpenAI                          |

### Configuración del entorno

```bash
# 1. Crear entorno virtual exclusivo para este lab
python -m venv venv_lab05
source venv_lab05/bin/activate        # Linux/macOS
# venv_lab05\Scripts\activate         # Windows

# 2. Instalar dependencias
pip install fastapi==0.115.0 \
            uvicorn[standard]==0.30.6 \
            pydantic==2.8.2 \
            "python-jose[cryptography]==3.3.0" \
            langgraph==0.2.28 \
            langchain==0.3.7 \
            langchain-openai==0.2.5 \
            httpx==0.27.2 \
            openai==1.40.6

# 3. Verificar instalaciones clave
python -c "import fastapi, langgraph, jose, pydantic; print('OK')"

# 4. Crear estructura del proyecto
mkdir -p lab05/{server,agent,tests,data}
cd lab05
```

---

## Paso a Paso

---

### Paso 1 — Inicializar la Base de Datos SQLite de Productos

**Objetivo:** Crear la base de datos SQLite que la tool de consulta de productos usará como fuente de datos, con datos sintéticos de prueba.

#### Instrucciones

1. Crea el archivo `lab05/data/init_db.py`:

```python
# lab05/data/init_db.py
"""
Script de inicialización de la base de datos SQLite de productos.
Todos los datos son sintéticos y generados para propósitos de prueba.
"""
import sqlite3
import os

DB_PATH = os.path.join(os.path.dirname(__file__), "productos.db")

SCHEMA = """
CREATE TABLE IF NOT EXISTS productos (
    id          TEXT PRIMARY KEY,
    nombre      TEXT NOT NULL,
    categoria   TEXT NOT NULL,
    precio      REAL NOT NULL,
    stock       INTEGER NOT NULL,
    descripcion TEXT
);
"""

DATOS_INICIALES = [
    ("P-001", "Laptop UltraBook X1",   "Computadoras", 1299.99, 45, "Laptop empresarial 14 pulgadas, 16GB RAM"),
    ("P-002", "Monitor 27 4K",          "Periféricos",  449.00,  30, "Monitor 4K IPS con panel antirreflejo"),
    ("P-003", "Teclado Mecánico Pro",   "Periféricos",   89.99, 120, "Teclado mecánico retroiluminado RGB"),
    ("P-004", "Mouse Ergonómico V3",    "Periféricos",   59.99, 200, "Mouse inalámbrico ergonómico 3000 DPI"),
    ("P-005", "Switch 24 puertos",      "Redes",        389.00,  15, "Switch gestionable Gigabit 24p"),
    ("P-006", "Servidor Rack 2U",       "Servidores",  3499.00,   8, "Servidor 2U con 2x Xeon, 64GB ECC"),
    ("P-007", "UPS 1500VA",             "Energía",      229.00,  50, "UPS de línea interactiva con LCD"),
    ("P-008", "Cable Cat6 100m",        "Redes",         35.50, 300, "Bobina cable Cat6 FTP 100 metros"),
]


def inicializar_db():
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute(SCHEMA)
    cursor.executemany(
        "INSERT OR IGNORE INTO productos VALUES (?,?,?,?,?,?)",
        DATOS_INICIALES
    )
    conn.commit()
    conn.close()
    print(f"[DB] Base de datos inicializada en: {DB_PATH}")
    print(f"[DB] {len(DATOS_INICIALES)} productos de prueba insertados.")


if __name__ == "__main__":
    inicializar_db()
```

2. Ejecuta el script:

```bash
cd lab05
python data/init_db.py
```

#### Salida esperada

```
[DB] Base de datos inicializada en: /ruta/lab05/data/productos.db
[DB] 8 productos de prueba insertados.
```

#### Verificación

```bash
python -c "
import sqlite3
conn = sqlite3.connect('data/productos.db')
rows = conn.execute('SELECT id, nombre, precio FROM productos').fetchall()
for r in rows: print(r)
conn.close()
"
```

Deberías ver los 8 productos listados con su ID, nombre y precio.

---

### Paso 2 — Implementar el Módulo de Autenticación JWT

**Objetivo:** Crear el módulo de autenticación que genera y valida tokens JWT con scopes diferenciados, siguiendo el patrón OAuth2 Bearer Token.

#### Instrucciones

1. Crea el archivo `lab05/server/auth.py`:

```python
# lab05/server/auth.py
"""
Módulo de autenticación JWT con scopes para el servidor MCP.

Scopes definidos:
  - tools:productos   → acceso a la tool de consulta de productos
  - tools:notificaciones → acceso a la tool de envío de notificaciones
  - tools:admin       → acceso completo a todas las tools
"""
from datetime import datetime, timedelta, timezone
from typing import Optional

from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt
from pydantic import BaseModel

# ─── Configuración de seguridad ───────────────────────────────────────────────
SECRET_KEY = "mcp-lab05-secret-key-NO-usar-en-produccion-2025"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

# Usuarios de prueba con sus scopes (en producción usar base de datos)
USUARIOS_PRUEBA = {
    "agente_productos": {
        "password": "pass_productos_123",
        "scopes": ["tools:productos"],
    },
    "agente_notificaciones": {
        "password": "pass_notif_456",
        "scopes": ["tools:notificaciones"],
    },
    "agente_admin": {
        "password": "pass_admin_789",
        "scopes": ["tools:productos", "tools:notificaciones", "tools:admin"],
    },
}

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")


# ─── Modelos Pydantic ──────────────────────────────────────────────────────────
class Token(BaseModel):
    access_token: str
    token_type: str
    expires_in: int
    scopes: list[str]


class TokenData(BaseModel):
    username: Optional[str] = None
    scopes: list[str] = []


# ─── Funciones de utilidad ────────────────────────────────────────────────────
def crear_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    """Genera un JWT firmado con los datos y scopes proporcionados."""
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + (
        expires_delta or timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    )
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)


def autenticar_usuario(username: str, password: str) -> Optional[dict]:
    """Valida credenciales y retorna el usuario si son correctas."""
    user = USUARIOS_PRUEBA.get(username)
    if not user or user["password"] != password:
        return None
    return {"username": username, **user}


async def obtener_token_data(token: str = Depends(oauth2_scheme)) -> TokenData:
    """Dependencia FastAPI: valida el JWT y extrae los datos del token."""
    credenciales_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="No se pudo validar el token de acceso",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credenciales_exception
        scopes: list[str] = payload.get("scopes", [])
        return TokenData(username=username, scopes=scopes)
    except JWTError:
        raise credenciales_exception


def verificar_scope(scope_requerido: str):
    """
    Factoría de dependencias: retorna una dependencia FastAPI que verifica
    que el token contenga el scope requerido.
    """
    async def _verificar(token_data: TokenData = Depends(obtener_token_data)):
        if scope_requerido not in token_data.scopes and "tools:admin" not in token_data.scopes:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Scope insuficiente. Se requiere: '{scope_requerido}'",
                headers={"WWW-Authenticate": f'Bearer scope="{scope_requerido}"'},
            )
        return token_data
    return _verificar
```

#### Salida esperada

No hay salida directa. Verificación sintáctica:

```bash
python -c "from server.auth import crear_access_token, verificar_scope; print('Auth module OK')"
```

#### Verificación

```bash
# Desde el directorio lab05
python -c "
from server.auth import crear_access_token, autenticar_usuario
user = autenticar_usuario('agente_admin', 'pass_admin_789')
print('Usuario autenticado:', user['username'])
token = crear_access_token({'sub': user['username'], 'scopes': user['scopes']})
print('Token generado (primeros 50 chars):', token[:50], '...')
"
```

---

### Paso 3 — Implementar las Tools del Servidor MCP

**Objetivo:** Crear los esquemas Pydantic y la lógica de negocio de las dos tools: consulta de productos y envío de notificaciones.

#### Instrucciones

1. Crea el archivo `lab05/server/tools.py`:

```python
# lab05/server/tools.py
"""
Lógica de negocio de las tools expuestas por el servidor MCP.

Tool 1: consultar_productos  (scope: tools:productos)
Tool 2: enviar_notificacion  (scope: tools:notificaciones)
"""
import sqlite3
import os
from datetime import datetime, timezone
from typing import Optional, Literal
from pydantic import BaseModel, Field, field_validator

DB_PATH = os.path.join(os.path.dirname(__file__), "../data/productos.db")


# ══════════════════════════════════════════════════════════════════════════════
# TOOL 1: Consulta de Productos
# ══════════════════════════════════════════════════════════════════════════════

class ConsultaProductosRequest(BaseModel):
    """Esquema de entrada para la tool de consulta de productos."""
    categoria: Optional[str] = Field(
        default=None,
        description="Filtrar por categoría (ej: 'Computadoras', 'Periféricos')",
        examples=["Computadoras"]
    )
    precio_maximo: Optional[float] = Field(
        default=None,
        ge=0,
        description="Precio máximo en USD (debe ser >= 0)",
        examples=[500.0]
    )
    stock_minimo: Optional[int] = Field(
        default=None,
        ge=0,
        description="Stock mínimo disponible",
        examples=[10]
    )
    id_producto: Optional[str] = Field(
        default=None,
        description="ID exacto del producto (ej: 'P-001')",
        examples=["P-001"]
    )

    @field_validator("id_producto")
    @classmethod
    def validar_formato_id(cls, v: Optional[str]) -> Optional[str]:
        if v is not None and not v.startswith("P-"):
            raise ValueError("El ID de producto debe tener formato 'P-XXX'")
        return v


class ProductoItem(BaseModel):
    """Representación de un producto en la respuesta."""
    id: str
    nombre: str
    categoria: str
    precio: float
    stock: int
    descripcion: Optional[str] = None


class ConsultaProductosResponse(BaseModel):
    """Esquema de respuesta para la tool de consulta de productos."""
    total_encontrados: int
    productos: list[ProductoItem]
    filtros_aplicados: dict
    timestamp: str


def ejecutar_consulta_productos(req: ConsultaProductosRequest) -> ConsultaProductosResponse:
    """
    Ejecuta la consulta en SQLite con los filtros proporcionados.
    Retorna los productos que coinciden con los criterios.
    """
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    cursor = conn.cursor()

    query = "SELECT * FROM productos WHERE 1=1"
    params = []
    filtros = {}

    if req.id_producto:
        query += " AND id = ?"
        params.append(req.id_producto)
        filtros["id_producto"] = req.id_producto

    if req.categoria:
        query += " AND categoria = ?"
        params.append(req.categoria)
        filtros["categoria"] = req.categoria

    if req.precio_maximo is not None:
        query += " AND precio <= ?"
        params.append(req.precio_maximo)
        filtros["precio_maximo"] = req.precio_maximo

    if req.stock_minimo is not None:
        query += " AND stock >= ?"
        params.append(req.stock_minimo)
        filtros["stock_minimo"] = req.stock_minimo

    rows = cursor.execute(query, params).fetchall()
    conn.close()

    productos = [
        ProductoItem(
            id=row["id"],
            nombre=row["nombre"],
            categoria=row["categoria"],
            precio=row["precio"],
            stock=row["stock"],
            descripcion=row["descripcion"],
        )
        for row in rows
    ]

    return ConsultaProductosResponse(
        total_encontrados=len(productos),
        productos=productos,
        filtros_aplicados=filtros if filtros else {"ninguno": "todos los productos"},
        timestamp=datetime.now(timezone.utc).isoformat(),
    )


# ══════════════════════════════════════════════════════════════════════════════
# TOOL 2: Envío de Notificaciones (Mock)
# ══════════════════════════════════════════════════════════════════════════════

class EnviarNotificacionRequest(BaseModel):
    """Esquema de entrada para la tool de envío de notificaciones."""
    canal: Literal["email", "slack", "sms"] = Field(
        description="Canal de notificación destino",
        examples=["email"]
    )
    destinatario: str = Field(
        min_length=3,
        max_length=200,
        description="Dirección email, canal Slack (#canal) o número telefónico",
        examples=["equipo-ventas@empresa.com"]
    )
    asunto: str = Field(
        min_length=5,
        max_length=200,
        description="Asunto o título de la notificación",
        examples=["Alerta de stock bajo"]
    )
    mensaje: str = Field(
        min_length=10,
        max_length=2000,
        description="Cuerpo del mensaje de notificación",
        examples=["El producto P-001 tiene menos de 10 unidades en stock."]
    )
    prioridad: Literal["baja", "normal", "alta", "critica"] = Field(
        default="normal",
        description="Nivel de prioridad de la notificación"
    )

    @field_validator("destinatario")
    @classmethod
    def validar_destinatario(cls, v: str, info) -> str:
        # Validación básica según canal (se evalúa en conjunto con canal)
        if "@" not in v and not v.startswith("#") and not v.startswith("+"):
            raise ValueError(
                "El destinatario debe ser un email (con @), canal Slack (con #) "
                "o número telefónico (con +)"
            )
        return v


class EnviarNotificacionResponse(BaseModel):
    """Esquema de respuesta para la tool de envío de notificaciones."""
    enviado: bool
    id_notificacion: str
    canal: str
    destinatario: str
    timestamp: str
    mensaje_confirmacion: str


def ejecutar_envio_notificacion(req: EnviarNotificacionRequest) -> EnviarNotificacionResponse:
    """
    Simula el envío de una notificación (mock).
    En producción, aquí iría la integración real con el servicio de mensajería.
    """
    import uuid
    notif_id = f"NOTIF-{uuid.uuid4().hex[:8].upper()}"

    # Simulación de lógica de envío por canal
    mensajes_confirmacion = {
        "email": f"Email enviado a {req.destinatario} con asunto '{req.asunto}'",
        "slack": f"Mensaje publicado en canal Slack {req.destinatario}",
        "sms": f"SMS enviado al número {req.destinatario}",
    }

    return EnviarNotificacionResponse(
        enviado=True,
        id_notificacion=notif_id,
        canal=req.canal,
        destinatario=req.destinatario,
        timestamp=datetime.now(timezone.utc).isoformat(),
        mensaje_confirmacion=mensajes_confirmacion[req.canal],
    )
```

#### Verificación

```bash
python -c "
import sys; sys.path.insert(0, '.')
from server.tools import ConsultaProductosRequest, ejecutar_consulta_productos
req = ConsultaProductosRequest(categoria='Periféricos')
resp = ejecutar_consulta_productos(req)
print(f'Productos encontrados: {resp.total_encontrados}')
for p in resp.productos:
    print(f'  {p.id} - {p.nombre} - \${p.precio}')
"
```

Deberías ver 3 productos de la categoría "Periféricos".

---

### Paso 4 — Construir el Servidor MCP con FastAPI

**Objetivo:** Ensamblar el servidor MCP completo combinando autenticación JWT, las tools implementadas y la documentación OpenAPI automática.

#### Instrucciones

1. Crea el archivo `lab05/server/main.py`:

```python
# lab05/server/main.py
"""
Servidor MCP implementado con FastAPI.
Expone dos tools de negocio con autenticación JWT y scopes.

Endpoints principales:
  POST /auth/token              → Obtener token JWT
  POST /mcp/tools/list          → Listar tools disponibles (requiere auth)
  POST /mcp/tools/call          → Invocar una tool (requiere auth + scope)
  GET  /docs                    → Swagger UI (documentación automática)
  GET  /health                  → Health check (público)
"""
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Any, Optional
from datetime import timedelta

# Importar módulos del servidor MCP
import sys, os
sys.path.insert(0, os.path.dirname(os.path.dirname(__file__)))

from server.auth import (
    Token, TokenData,
    autenticar_usuario, crear_access_token,
    obtener_token_data, verificar_scope,
    ACCESS_TOKEN_EXPIRE_MINUTES,
)
from server.tools import (
    ConsultaProductosRequest, ConsultaProductosResponse, ejecutar_consulta_productos,
    EnviarNotificacionRequest, EnviarNotificacionResponse, ejecutar_envio_notificacion,
)

# ─── Inicialización de la aplicación ─────────────────────────────────────────
app = FastAPI(
    title="Servidor MCP — Lab 05",
    description="""
## Servidor Model Context Protocol (MCP)

Implementación de referencia de un servidor MCP con FastAPI.
Expone tools de negocio con autenticación JWT y scopes diferenciados.

### Scopes disponibles
- `tools:productos` — Acceso a consulta de productos
- `tools:notificaciones` — Acceso a envío de notificaciones
- `tools:admin` — Acceso completo a todas las tools

### Usuarios de prueba
| Usuario               | Password           | Scopes                              |
|-----------------------|--------------------|-------------------------------------|
| agente_productos      | pass_productos_123 | tools:productos                     |
| agente_notificaciones | pass_notif_456     | tools:notificaciones                |
| agente_admin          | pass_admin_789     | tools:productos, tools:notificaciones, tools:admin |
    """,
    version="1.0.0",
    contact={"name": "Lab MCP", "email": "lab@ejemplo.com"},
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# ─── Esquemas MCP genéricos ───────────────────────────────────────────────────

class MCPToolDefinition(BaseModel):
    name: str
    description: str
    input_schema: dict
    required_scope: str


class MCPToolsListResponse(BaseModel):
    tools: list[MCPToolDefinition]
    server_version: str = "1.0.0"


class MCPToolCallRequest(BaseModel):
    name: str
    arguments: dict[str, Any]


class MCPToolCallResponse(BaseModel):
    tool_name: str
    success: bool
    result: Optional[Any] = None
    error: Optional[str] = None


# Definición estática del catálogo de tools
TOOLS_CATALOG: list[MCPToolDefinition] = [
    MCPToolDefinition(
        name="consultar_productos",
        description=(
            "Consulta el catálogo de productos en la base de datos. "
            "Permite filtrar por categoría, precio máximo, stock mínimo o ID específico."
        ),
        input_schema=ConsultaProductosRequest.model_json_schema(),
        required_scope="tools:productos",
    ),
    MCPToolDefinition(
        name="enviar_notificacion",
        description=(
            "Envía una notificación a través del canal especificado (email, Slack o SMS). "
            "Requiere destinatario, asunto y mensaje."
        ),
        input_schema=EnviarNotificacionRequest.model_json_schema(),
        required_scope="tools:notificaciones",
    ),
]


# ─── Endpoints de autenticación ───────────────────────────────────────────────

@app.post(
    "/auth/token",
    response_model=Token,
    summary="Obtener token de acceso JWT",
    tags=["Autenticación"],
)
async def login_para_token(form_data: OAuth2PasswordRequestForm = Depends()):
    """
    Autentica al usuario y retorna un JWT con los scopes asociados.
    Usa el formulario OAuth2 estándar (username + password).
    """
    usuario = autenticar_usuario(form_data.username, form_data.password)
    if not usuario:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Credenciales incorrectas",
            headers={"WWW-Authenticate": "Bearer"},
        )
    access_token = crear_access_token(
        data={"sub": usuario["username"], "scopes": usuario["scopes"]},
        expires_delta=timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES),
    )
    return Token(
        access_token=access_token,
        token_type="bearer",
        expires_in=ACCESS_TOKEN_EXPIRE_MINUTES * 60,
        scopes=usuario["scopes"],
    )


# ─── Endpoints MCP ────────────────────────────────────────────────────────────

@app.get(
    "/health",
    summary="Health check del servidor MCP",
    tags=["Sistema"],
)
async def health_check():
    """Endpoint público para verificar que el servidor está activo."""
    return {"status": "healthy", "server": "MCP Lab 05", "version": "1.0.0"}


@app.post(
    "/mcp/tools/list",
    response_model=MCPToolsListResponse,
    summary="Listar tools disponibles en el servidor MCP",
    tags=["MCP Protocol"],
)
async def listar_tools(token_data: TokenData = Depends(obtener_token_data)):
    """
    Retorna el catálogo de tools disponibles con sus esquemas JSON.
    Requiere un token JWT válido (cualquier scope).
    Equivalente al método `tools/list` del protocolo MCP.
    """
    # Filtrar tools según los scopes del token
    tools_accesibles = [
        tool for tool in TOOLS_CATALOG
        if tool.required_scope in token_data.scopes
        or "tools:admin" in token_data.scopes
    ]
    return MCPToolsListResponse(tools=tools_accesibles)


@app.post(
    "/mcp/tools/call",
    response_model=MCPToolCallResponse,
    summary="Invocar una tool del servidor MCP",
    tags=["MCP Protocol"],
    responses={
        200: {"description": "Tool ejecutada exitosamente"},
        400: {"description": "Argumentos inválidos para la tool"},
        401: {"description": "Token no proporcionado o inválido"},
        403: {"description": "Scope insuficiente para esta tool"},
        404: {"description": "Tool no encontrada"},
    },
)
async def invocar_tool(
    request: MCPToolCallRequest,
    token_data: TokenData = Depends(obtener_token_data),
):
    """
    Invoca una tool por nombre con los argumentos proporcionados.
    Verifica el scope requerido antes de ejecutar.
    Equivalente al método `tools/call` del protocolo MCP.
    """
    # Buscar la tool en el catálogo
    tool_def = next((t for t in TOOLS_CATALOG if t.name == request.name), None)
    if not tool_def:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Tool '{request.name}' no encontrada en este servidor MCP",
        )

    # Verificar scope
    if (
        tool_def.required_scope not in token_data.scopes
        and "tools:admin" not in token_data.scopes
    ):
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail=f"Scope insuficiente. Se requiere '{tool_def.required_scope}' "
                   f"para invocar '{request.name}'",
        )

    # Ejecutar la tool correspondiente
    try:
        if request.name == "consultar_productos":
            req_model = ConsultaProductosRequest(**request.arguments)
            result = ejecutar_consulta_productos(req_model)
            return MCPToolCallResponse(
                tool_name=request.name,
                success=True,
                result=result.model_dump(),
            )

        elif request.name == "enviar_notificacion":
            req_model = EnviarNotificacionRequest(**request.arguments)
            result = ejecutar_envio_notificacion(req_model)
            return MCPToolCallResponse(
                tool_name=request.name,
                success=True,
                result=result.model_dump(),
            )

    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=f"Error al ejecutar tool '{request.name}': {str(e)}",
        )
```

2. Crea el archivo `lab05/__init__.py` y `lab05/server/__init__.py` (vacíos):

```bash
touch lab05/__init__.py lab05/server/__init__.py lab05/agent/__init__.py lab05/tests/__init__.py
```

3. Inicia el servidor:

```bash
cd lab05
uvicorn server.main:app --host 0.0.0.0 --port 8000 --reload
```

#### Salida esperada

```
INFO:     Will watch for changes in these directories: ['/ruta/lab05']
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [XXXXX] using WatchFiles
INFO:     Started server process [XXXXX]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
```

#### Verificación

Abre un **segundo terminal** y ejecuta:

```bash
# Health check
curl -s http://localhost:8000/health | python -m json.tool

# Verificar que Swagger UI está disponible
curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/docs
# Debe retornar: 200
```

También puedes abrir `http://localhost:8000/docs` en el navegador para explorar la documentación interactiva.

---

### Paso 5 — Pruebas de Seguridad del Servidor MCP

**Objetivo:** Verificar los controles de autenticación y autorización ejecutando pruebas de acceso sin token, con token inválido y con scopes insuficientes.

#### Instrucciones

1. Crea el archivo `lab05/tests/test_seguridad.py`:

```python
# lab05/tests/test_seguridad.py
"""
Pruebas de seguridad del servidor MCP.
Verifica controles de autenticación y autorización por scopes.

IMPORTANTE: Estas pruebas requieren el servidor corriendo en localhost:8000
"""
import httpx
import json

BASE_URL = "http://localhost:8000"


def obtener_token(username: str, password: str) -> str:
    """Helper: obtiene un JWT para el usuario especificado."""
    resp = httpx.post(
        f"{BASE_URL}/auth/token",
        data={"username": username, "password": password},
    )
    assert resp.status_code == 200, f"Error al obtener token: {resp.text}"
    return resp.json()["access_token"]


def separador(titulo: str):
    print(f"\n{'='*60}")
    print(f"  {titulo}")
    print('='*60)


# ─── PRUEBA 1: Acceso sin token ───────────────────────────────────────────────
def test_acceso_sin_token():
    separador("PRUEBA 1: Acceso sin token")
    resp = httpx.post(
        f"{BASE_URL}/mcp/tools/list",
        json={},
    )
    print(f"Status: {resp.status_code} (esperado: 401)")
    print(f"Respuesta: {resp.json()}")
    assert resp.status_code == 401, f"FALLO: Se esperaba 401, se obtuvo {resp.status_code}"
    print("✅ PASÓ: Acceso rechazado correctamente sin token")


# ─── PRUEBA 2: Token con formato inválido ────────────────────────────────────
def test_token_invalido():
    separador("PRUEBA 2: Token con formato inválido")
    resp = httpx.post(
        f"{BASE_URL}/mcp/tools/list",
        headers={"Authorization": "Bearer token_falso_no_valido_123"},
        json={},
    )
    print(f"Status: {resp.status_code} (esperado: 401)")
    print(f"Respuesta: {resp.json()}")
    assert resp.status_code == 401, f"FALLO: Se esperaba 401, se obtuvo {resp.status_code}"
    print("✅ PASÓ: Token inválido rechazado correctamente")


# ─── PRUEBA 3: Scope insuficiente ────────────────────────────────────────────
def test_scope_insuficiente():
    separador("PRUEBA 3: Scope insuficiente (agente_productos intenta enviar notificación)")
    token = obtener_token("agente_productos", "pass_productos_123")
    resp = httpx.post(
        f"{BASE_URL}/mcp/tools/call",
        headers={"Authorization": f"Bearer {token}"},
        json={
            "name": "enviar_notificacion",
            "arguments": {
                "canal": "email",
                "destinatario": "test@ejemplo.com",
                "asunto": "Test",
                "mensaje": "Mensaje de prueba de scope insuficiente",
            },
        },
    )
    print(f"Status: {resp.status_code} (esperado: 403)")
    print(f"Respuesta: {resp.json()}")
    assert resp.status_code == 403, f"FALLO: Se esperaba 403, se obtuvo {resp.status_code}"
    print("✅ PASÓ: Scope insuficiente rechazado correctamente")


# ─── PRUEBA 4: Tool no existente ─────────────────────────────────────────────
def test_tool_inexistente():
    separador("PRUEBA 4: Invocar tool que no existe")
    token = obtener_token("agente_admin", "pass_admin_789")
    resp = httpx.post(
        f"{BASE_URL}/mcp/tools/call",
        headers={"Authorization": f"Bearer {token}"},
        json={"name": "tool_que_no_existe", "arguments": {}},
    )
    print(f"Status: {resp.status_code} (esperado: 404)")
    print(f"Respuesta: {resp.json()}")
    assert resp.status_code == 404, f"FALLO: Se esperaba 404, se obtuvo {resp.status_code}"
    print("✅ PASÓ: Tool inexistente retorna 404 correctamente")


# ─── PRUEBA 5: Flujo exitoso con admin ───────────────────────────────────────
def test_flujo_exitoso_admin():
    separador("PRUEBA 5: Flujo exitoso con agente_admin")
    token = obtener_token("agente_admin", "pass_admin_789")

    # Listar tools
    resp_list = httpx.post(
        f"{BASE_URL}/mcp/tools/list",
        headers={"Authorization": f"Bearer {token}"},
        json={},
    )
    assert resp_list.status_code == 200
    tools = resp_list.json()["tools"]
    print(f"Tools disponibles para admin: {[t['name'] for t in tools]}")
    assert len(tools) == 2, f"Se esperaban 2 tools, se obtuvieron {len(tools)}"

    # Invocar tool de productos
    resp_call = httpx.post(
        f"{BASE_URL}/mcp/tools/call",
        headers={"Authorization": f"Bearer {token}"},
        json={
            "name": "consultar_productos",
            "arguments": {"categoria": "Periféricos", "stock_minimo": 10},
        },
    )
    assert resp_call.status_code == 200
    result = resp_call.json()
    print(f"Productos encontrados: {result['result']['total_encontrados']}")
    assert result["success"] is True
    print("✅ PASÓ: Flujo completo exitoso con admin")


# ─── PRUEBA 6: Validación de esquema Pydantic ────────────────────────────────
def test_validacion_esquema():
    separador("PRUEBA 6: Argumentos inválidos (validación Pydantic)")
    token = obtener_token("agente_admin", "pass_admin_789")
    resp = httpx.post(
        f"{BASE_URL}/mcp/tools/call",
        headers={"Authorization": f"Bearer {token}"},
        json={
            "name": "consultar_productos",
            "arguments": {"precio_maximo": -50},  # precio negativo: inválido
        },
    )
    print(f"Status: {resp.status_code} (esperado: 400)")
    print(f"Respuesta: {resp.json()}")
    assert resp.status_code == 400, f"FALLO: Se esperaba 400, se obtuvo {resp.status_code}"
    print("✅ PASÓ: Validación de esquema funciona correctamente")


if __name__ == "__main__":
    print("\n🔐 INICIANDO PRUEBAS DE SEGURIDAD DEL SERVIDOR MCP")
    print("Servidor esperado en:", BASE_URL)

    test_acceso_sin_token()
    test_token_invalido()
    test_scope_insuficiente()
    test_tool_inexistente()
    test_flujo_exitoso_admin()
    test_validacion_esquema()

    print("\n" + "="*60)
    print("  ✅ TODAS LAS PRUEBAS DE SEGURIDAD PASARON")
    print("="*60)
```

2. Ejecuta las pruebas (con el servidor corriendo en otro terminal):

```bash
cd lab05
python tests/test_seguridad.py
```

#### Salida esperada

```
🔐 INICIANDO PRUEBAS DE SEGURIDAD DEL SERVIDOR MCP
Servidor esperado en: http://localhost:8000

============================================================
  PRUEBA 1: Acceso sin token
============================================================
Status: 401 (esperado: 401)
Respuesta: {'detail': 'Not authenticated'}
✅ PASÓ: Acceso rechazado correctamente sin token

============================================================
  PRUEBA 2: Token con formato inválido
============================================================
Status: 401 (esperado: 401)
Respuesta: {'detail': 'No se pudo validar el token de acceso'}
✅ PASÓ: Token inválido rechazado correctamente

============================================================
  PRUEBA 3: Scope insuficiente (agente_productos intenta enviar notificación)
============================================================
Status: 403 (esperado: 403)
Respuesta: {'detail': "Scope insuficiente. Se requiere 'tools:notificaciones' para invocar 'enviar_notificacion'"}
✅ PASÓ: Scope insuficiente rechazado correctamente

[... pruebas 4, 5, 6 ...]

============================================================
  ✅ TODAS LAS PRUEBAS DE SEGURIDAD PASARON
============================================================
```

#### Verificación

Todas las pruebas deben mostrar `✅ PASÓ`. Si alguna falla, revisa que el servidor esté corriendo y que la base de datos esté inicializada.

---

### Paso 6 — Implementar el Cliente MCP para LangGraph

**Objetivo:** Crear el módulo cliente que traduce las tools del servidor MCP en herramientas LangChain/LangGraph, permitiendo que el agente las invoque de forma transparente.

#### Instrucciones

1. Crea el archivo `lab05/agent/mcp_client.py`:

```python
# lab05/agent/mcp_client.py
"""
Cliente MCP que conecta el agente LangGraph con el servidor MCP.
Traduce las tools del servidor MCP en herramientas LangChain.
"""
import httpx
from typing import Any
from langchain_core.tools import tool
from functools import partial

MCP_BASE_URL = "http://localhost:8000"
_token_cache: dict[str, str] = {}


class MCPClient:
    """
    Cliente HTTP para interactuar con el servidor MCP.
    Gestiona autenticación JWT y la invocación de tools.
    """

    def __init__(self, base_url: str = MCP_BASE_URL):
        self.base_url = base_url
        self._token: str | None = None
        self._client = httpx.Client(timeout=30.0)

    def autenticar(self, username: str, password: str) -> str:
        """Obtiene y cachea el token JWT del servidor MCP."""
        resp = self._client.post(
            f"{self.base_url}/auth/token",
            data={"username": username, "password": password},
        )
        resp.raise_for_status()
        self._token = resp.json()["access_token"]
        scopes = resp.json()["scopes"]
        print(f"[MCP Client] Autenticado como '{username}' con scopes: {scopes}")
        return self._token

    def _headers(self) -> dict:
        if not self._token:
            raise RuntimeError("MCPClient no autenticado. Llama a autenticar() primero.")
        return {"Authorization": f"Bearer {self._token}"}

    def listar_tools(self) -> list[dict]:
        """Descubre las tools disponibles en el servidor MCP."""
        resp = self._client.post(
            f"{self.base_url}/mcp/tools/list",
            headers=self._headers(),
            json={},
        )
        resp.raise_for_status()
        return resp.json()["tools"]

    def invocar_tool(self, nombre: str, argumentos: dict) -> Any:
        """Invoca una tool del servidor MCP y retorna el resultado."""
        resp = self._client.post(
            f"{self.base_url}/mcp/tools/call",
            headers=self._headers(),
            json={"name": nombre, "arguments": argumentos},
        )
        if resp.status_code == 403:
            raise PermissionError(f"Scope insuficiente para tool '{nombre}': {resp.json()['detail']}")
        if resp.status_code == 404:
            raise ValueError(f"Tool '{nombre}' no encontrada en el servidor MCP")
        resp.raise_for_status()
        result = resp.json()
        if not result["success"]:
            raise RuntimeError(f"Error en tool '{nombre}': {result.get('error')}")
        return result["result"]

    def cerrar(self):
        self._client.close()


def crear_tools_desde_mcp(mcp_client: MCPClient) -> list:
    """
    Descubre las tools del servidor MCP y las convierte en herramientas
    LangChain/LangGraph que el agente puede usar directamente.
    """
    tools_mcp = mcp_client.listar_tools()
    langchain_tools = []

    for tool_def in tools_mcp:
        nombre = tool_def["name"]
        descripcion = tool_def["description"]

        # Crear una función dinámica que invoque esta tool en el servidor MCP
        def _crear_invocador(tool_nombre: str, tool_descripcion: str):
            def _invocar(**kwargs) -> str:
                """Invoca la tool en el servidor MCP y formatea el resultado."""
                try:
                    resultado = mcp_client.invocar_tool(tool_nombre, kwargs)
                    import json
                    return json.dumps(resultado, ensure_ascii=False, indent=2)
                except PermissionError as e:
                    return f"ERROR DE AUTORIZACIÓN: {str(e)}"
                except Exception as e:
                    return f"ERROR AL INVOCAR TOOL '{tool_nombre}': {str(e)}"

            _invocar.__name__ = tool_nombre
            _invocar.__doc__ = tool_descripcion
            return _invocar

        invocar_fn = _crear_invocador(nombre, descripcion)

        # Envolver en herramienta LangChain usando el decorador @tool
        lc_tool = tool(invocar_fn)
        langchain_tools.append(lc_tool)
        print(f"[MCP Client] Tool registrada: '{nombre}'")

    return langchain_tools
```

#### Verificación

```bash
# Con el servidor corriendo en otro terminal:
python -c "
import sys; sys.path.insert(0, '.')
from agent.mcp_client import MCPClient, crear_tools_desde_mcp
client = MCPClient()
client.autenticar('agente_admin', 'pass_admin_789')
tools = client.listar_tools()
print('Tools descubiertas:', [t['name'] for t in tools])
lc_tools = crear_tools_desde_mcp(client)
print('LangChain tools creadas:', [t.name for t in lc_tools])
client.cerrar()
"
```

Salida esperada:
```
[MCP Client] Autenticado como 'agente_admin' con scopes: ['tools:productos', 'tools:notificaciones', 'tools:admin']
Tools descubiertas: ['consultar_productos', 'enviar_notificacion']
[MCP Client] Tool registrada: 'consultar_productos'
[MCP Client] Tool registrada: 'enviar_notificacion'
LangChain tools creadas: ['consultar_productos', 'enviar_notificacion']
```

---

### Paso 7 — Construir el Agente LangGraph Conectado al Servidor MCP

**Objetivo:** Reconfigurar el agente LangGraph del Lab 4 para consumir las tools desde el servidor MCP en lugar de herramientas locales, verificando el flujo completo de invocación.

#### Instrucciones

1. Crea el archivo `lab05/agent/agente_mcp.py`:

```python
# lab05/agent/agente_mcp.py
"""
Agente LangGraph que consume tools desde el servidor MCP.
Implementa el patrón ReAct usando tools descubiertas dinámicamente.

PREREQUISITO: El servidor MCP debe estar corriendo en localhost:8000
"""
import os
from typing import Annotated, TypedDict
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode

# Importar el cliente MCP
import sys
sys.path.insert(0, os.path.dirname(os.path.dirname(__file__)))
from agent.mcp_client import MCPClient, crear_tools_desde_mcp

# ─── Configuración del agente ─────────────────────────────────────────────────
OPENAI_API_KEY = os.environ.get("OPENAI_API_KEY")
if not OPENAI_API_KEY:
    raise EnvironmentError("Variable OPENAI_API_KEY no configurada")

MCP_USERNAME = "agente_admin"
MCP_PASSWORD = "pass_admin_789"

SYSTEM_PROMPT = """Eres un asistente de gestión de inventario y comunicaciones.
Tienes acceso a las siguientes herramientas a través del servidor MCP:

1. **consultar_productos**: Busca productos en el catálogo por categoría, precio, stock o ID.
2. **enviar_notificacion**: Envía notificaciones por email, Slack o SMS.

Instrucciones:
- Usa las herramientas cuando el usuario solicite información de productos o quiera enviar notificaciones.
- Siempre reporta los resultados de forma clara y estructurada.
- Si hay un error, explícalo al usuario de forma comprensible.
- Para enviar notificaciones, usa siempre direcciones de prueba (test@ejemplo.com o #canal-test).
"""


# ─── Definición del estado del agente ────────────────────────────────────────
class AgenteState(TypedDict):
    messages: Annotated[list, lambda x, y: x + y]


# ─── Construcción del grafo LangGraph ────────────────────────────────────────
def construir_agente_mcp():
    """
    Construye y retorna el grafo LangGraph configurado para usar
    las tools del servidor MCP.
    """
    # 1. Inicializar el cliente MCP y obtener las tools
    print("[Agente] Conectando al servidor MCP...")
    mcp_client = MCPClient()
    mcp_client.autenticar(MCP_USERNAME, MCP_PASSWORD)
    tools = crear_tools_desde_mcp(mcp_client)
    print(f"[Agente] {len(tools)} tools cargadas desde el servidor MCP")

    # 2. Configurar el LLM con las tools vinculadas
    llm = ChatOpenAI(
        model="gpt-4o-mini",
        temperature=0,
        api_key=OPENAI_API_KEY,
    )
    llm_con_tools = llm.bind_tools(tools)

    # 3. Definir los nodos del grafo
    def nodo_razonamiento(state: AgenteState) -> dict:
        """Nodo de razonamiento: el LLM decide qué tool invocar o genera respuesta final."""
        messages = state["messages"]
        if not any(isinstance(m, SystemMessage) for m in messages):
            messages = [SystemMessage(content=SYSTEM_PROMPT)] + messages
        respuesta = llm_con_tools.invoke(messages)
        return {"messages": [respuesta]}

    def debe_continuar(state: AgenteState) -> str:
        """Router: determina si continuar con tools o terminar."""
        ultimo_mensaje = state["messages"][-1]
        if hasattr(ultimo_mensaje, "tool_calls") and ultimo_mensaje.tool_calls:
            return "ejecutar_tools"
        return END

    # 4. Construir el grafo
    tool_node = ToolNode(tools)
    grafo = StateGraph(AgenteState)
    grafo.add_node("razonamiento", nodo_razonamiento)
    grafo.add_node("ejecutar_tools", tool_node)
    grafo.set_entry_point("razonamiento")
    grafo.add_conditional_edges("razonamiento", debe_continuar)
    grafo.add_edge("ejecutar_tools", "razonamiento")

    return grafo.compile(), mcp_client


# ─── Función de ejecución interactiva ────────────────────────────────────────
def ejecutar_consulta(agente, consulta: str) -> str:
    """Ejecuta una consulta en el agente y retorna la respuesta final."""
    print(f"\n{'─'*60}")
    print(f"👤 Usuario: {consulta}")
    print('─'*60)

    estado_inicial = {"messages": [HumanMessage(content=consulta)]}
    estado_final = agente.invoke(estado_inicial)

    respuesta = estado_final["messages"][-1].content
    print(f"🤖 Agente: {respuesta}")
    return respuesta


# ─── Punto de entrada principal ───────────────────────────────────────────────
if __name__ == "__main__":
    print("="*60)
    print("  AGENTE MCP — Lab 05")
    print("  Conectando a servidor MCP en localhost:8000")
    print("="*60)

    agente, mcp_client = construir_agente_mcp()

    # Consultas de demostración
    consultas = [
        "¿Qué productos de la categoría Periféricos están disponibles?",
        "¿Cuáles son los productos con precio menor a $100 y al menos 50 unidades en stock?",
        "Busca el producto con ID P-006 y dime sus detalles.",
        "Envía una notificación por email a test@ejemplo.com informando que el producto "
        "P-007 (UPS 1500VA) tiene solo 50 unidades restantes. Usa asunto 'Alerta de Stock'.",
    ]

    for consulta in consultas:
        ejecutar_consulta(agente, consulta)

    mcp_client.cerrar()
    print("\n[Agente] Sesión finalizada. Conexión MCP cerrada.")
```

2. Ejecuta el agente (con el servidor MCP corriendo en otro terminal):

```bash
cd lab05
python agent/agente_mcp.py
```

#### Salida esperada

```
============================================================
  AGENTE MCP — Lab 05
  Conectando a servidor MCP en localhost:8000
============================================================
[Agente] Conectando al servidor MCP...
[MCP Client] Autenticado como 'agente_admin' con scopes: [...]
[MCP Client] Tool registrada: 'consultar_productos'
[MCP Client] Tool registrada: 'enviar_notificacion'
[Agente] 2 tools cargadas desde el servidor MCP

────────────────────────────────────────────────────────────
👤 Usuario: ¿Qué productos de la categoría Periféricos están disponibles?
────────────────────────────────────────────────────────────
🤖 Agente: Encontré 3 productos en la categoría **Periféricos**:

| ID    | Nombre               | Precio  | Stock |
|-------|----------------------|---------|-------|
| P-002 | Monitor 27 4K        | $449.00 | 30    |
| P-003 | Teclado Mecánico Pro | $89.99  | 120   |
| P-004 | Mouse Ergonómico V3  | $59.99  | 200   |

[... respuestas para las demás consultas ...]

[Agente] Sesión finalizada. Conexión MCP cerrada.
```

#### Verificación

Las cuatro consultas deben obtener respuestas coherentes. La última consulta debe mostrar la confirmación del envío de la notificación con un ID como `NOTIF-XXXXXXXX`.

---

## Validación y Pruebas Finales

### Prueba de integración completa

Ejecuta la siguiente secuencia de validación en un script final:

```bash
# Con el servidor corriendo, ejecutar en lab05/:
python -c "
import httpx, json

BASE = 'http://localhost:8000'

# 1. Obtener token
resp = httpx.post(f'{BASE}/auth/token', data={'username':'agente_admin','password':'pass_admin_789'})
token = resp.json()['access_token']
headers = {'Authorization': f'Bearer {token}'}
print('✅ Token obtenido:', resp.json()['scopes'])

# 2. Listar tools
resp = httpx.post(f'{BASE}/mcp/tools/list', headers=headers, json={})
tools = resp.json()['tools']
print(f'✅ Tools disponibles: {[t[\"name\"] for t in tools]}')

# 3. Consultar productos sin filtros
resp = httpx.post(f'{BASE}/mcp/tools/call', headers=headers,
    json={'name': 'consultar_productos', 'arguments': {}})
total = resp.json()['result']['total_encontrados']
print(f'✅ Total productos en catálogo: {total}')

# 4. Enviar notificación
resp = httpx.post(f'{BASE}/mcp/tools/call', headers=headers,
    json={'name': 'enviar_notificacion', 'arguments': {
        'canal': 'slack', 'destinatario': '#test-canal',
        'asunto': 'Validación Lab 05', 'mensaje': 'Prueba de integración exitosa'
    }})
notif_id = resp.json()['result']['id_notificacion']
print(f'✅ Notificación enviada con ID: {notif_id}')

# 5. Verificar Swagger
resp = httpx.get(f'{BASE}/openapi.json')
paths = list(resp.json()['paths'].keys())
print(f'✅ Endpoints documentados en OpenAPI: {paths}')

print()
print('🎉 VALIDACIÓN COMPLETA — Servidor MCP operativo')
"
```

### Lista de verificación final

| Componente | Verificación | Estado esperado |
|---|---|---|
| Servidor MCP | `GET /health` retorna 200 | ✅ |
| Autenticación | `POST /auth/token` genera JWT | ✅ |
| Tool de productos | Consulta con filtros retorna datos correctos | ✅ |
| Tool de notificaciones | Mock retorna ID de notificación | ✅ |
| Seguridad: sin token | Retorna 401 | ✅ |
| Seguridad: scope insuficiente | Retorna 403 | ✅ |
| Swagger UI | Accesible en `/docs` | ✅ |
| Agente LangGraph | Invoca tools MCP correctamente | ✅ |

---

## Solución de Problemas

### Problema 1: `ConnectionRefusedError` al ejecutar el agente o las pruebas

**Síntoma:**
```
httpx.ConnectError: [Errno 111] Connection refused
```
o
```
httpcore.ConnectError: All connection attempts failed
```

**Causa:** El servidor MCP no está corriendo o está escuchando en un puerto diferente. El agente y las pruebas asumen que el servidor está disponible en `http://localhost:8000`.

**Solución:**
```bash
# 1. Verificar si el servidor está corriendo
lsof -i :8000          # Linux/macOS
netstat -ano | findstr :8000   # Windows

# 2. Si no está corriendo, iniciarlo en un terminal separado:
cd lab05
uvicorn server.main:app --host 0.0.0.0 --port 8000 --reload

# 3. Verificar que responde:
curl http://localhost:8000/health

# 4. Si el puerto 8000 está ocupado, cambiar en server/main.py y agent/mcp_client.py:
uvicorn server.main:app --port 8001 --reload
# Luego actualizar MCP_BASE_URL = "http://localhost:8001" en mcp_client.py
```

---

### Problema 2: `sqlite3.OperationalError: no such table: productos`

**Síntoma:**
```
sqlite3.OperationalError: no such table: productos
```
La tool `consultar_productos` falla con error 400 al ser invocada, o la verificación del Paso 1 no encuentra la tabla.

**Causa:** El script `data/init_db.py` no fue ejecutado, o fue ejecutado desde un directorio diferente al esperado, generando el archivo `productos.db` en una ruta distinta a la que usa `tools.py`.

**Solución:**
```bash
# 1. Verificar la ubicación del archivo de base de datos
find . -name "productos.db" 2>/dev/null

# 2. Si no existe, ejecutar el script desde el directorio correcto
cd lab05
python data/init_db.py

# 3. Verificar que el archivo se creó en data/
ls -la data/productos.db

# 4. Verificar que la ruta en tools.py apunta correctamente
python -c "
import os
DB_PATH = os.path.join(os.path.dirname('server/tools.py'), '../data/productos.db')
print('Ruta calculada:', os.path.abspath(DB_PATH))
print('Existe:', os.path.exists(os.path.abspath(DB_PATH)))
"

# 5. Si la ruta es incorrecta, usar ruta absoluta en tools.py:
# DB_PATH = "/ruta/absoluta/a/lab05/data/productos.db"
```

---

## Limpieza del Entorno

```bash
# 1. Detener el servidor MCP (Ctrl+C en el terminal donde corre uvicorn)

# 2. Desactivar el entorno virtual
deactivate

# 3. (Opcional) Eliminar el entorno virtual si no se necesita más
rm -rf venv_lab05

# 4. (Opcional) Eliminar la base de datos de prueba
rm lab05/data/productos.db

# 5. Verificar que el puerto 8000 quedó libre
lsof -i :8000    # No debe mostrar resultados
```

---

## Resumen

En este lab construiste un sistema completo de integración basado en el **Model Context Protocol (MCP)**, aplicando los principios de separación de responsabilidades que define el protocolo:

| Componente construido | Tecnología | Función en la arquitectura MCP |
|---|---|---|
| Servidor MCP | FastAPI + Uvicorn | MCP Server: expone tools con contrato OpenAPI |
| Autenticación JWT | python-jose | Control de acceso con scopes por tool |
| Tool de productos | SQLite3 + Pydantic | Primitiva tipo Resource/Tool del servidor |
| Tool de notificaciones | Mock + Pydantic | Primitiva tipo Tool con efectos secundarios |
| Cliente MCP | httpx | MCP Client: descubre e invoca tools |
| Agente consumidor | LangGraph + OpenAI | MCP Host: orquesta el ciclo completo |
| Pruebas de seguridad | httpx | Validación de controles 401/403/404 |

### Conceptos clave aplicados

- **Descubrimiento dinámico**: el agente consulta `/mcp/tools/list` en tiempo de ejecución para conocer las tools disponibles, sin hardcodearlas.
- **Separación de responsabilidades**: el agente solo habla el protocolo MCP; los detalles de SQLite y notificaciones quedan encapsulados en el servidor.
- **Scopes diferenciados**: `tools:productos` y `tools:notificaciones` permiten control granular de acceso, siguiendo el principio de mínimo privilegio.
- **Contratos OpenAPI**: Swagger UI generado automáticamente por FastAPI documenta el contrato de la API, facilitando la integración de nuevos agentes consumidores.

### Recursos adicionales

- [Especificación oficial de MCP](https://modelcontextprotocol.io/docs)
- [FastAPI: Seguridad con OAuth2 y JWT](https://fastapi.tiangolo.com/tutorial/security/oauth2-jwt/)
- [LangGraph: ToolNode y herramientas personalizadas](https://langchain-ai.github.io/langgraph/how-tos/tool-calling/)
- [python-jose: documentación de JWT](https://python-jose.readthedocs.io/)

---
LAB_END---
