---
description: "MCP Python server: FastAPI + MongoDB + External JWT + OpenTelemetry"
alwaysApply: true
---

# MCP Server - Python

Production MCP server: Python 3.11+, FastAPI, MongoDB, Redis. External auth system handles JWT/RBAC.

## Quick Reference

**Stack:** FastAPI, Motor (MongoDB), Redis, Pydantic, structlog, OpenTelemetry, Prometheus
**Auth:** External JWT verification only (no user DB)
**DB:** MongoDB for resources/prompts/tools/logs
**Transports:** stdio, HTTP+SSE, WebSocket

## Core Standards

Type hints (Python 3.11+):
```python
async def process(data: dict[str, Any], cb: Callable | None = None) -> tuple[int, str]:
    ...
```

Async I/O always:
```python
# ✅ Good
async def fetch(id: str) -> Resource:
    async with aiohttp.ClientSession() as session:
        return await session.get(f"/api/{id}")

# ❌ Bad - blocks event loop
def fetch(id: str) -> Resource:
    return requests.get(f"/api/{id}")
```

Pydantic v2 validation:
```python
class MCPRequest(BaseModel):
    model_config = ConfigDict(strict=True)
    jsonrpc: Literal["2.0"] = "2.0"
    method: str = Field(min_length=1, max_length=100)
    params: dict[str, Any] = Field(default_factory=dict)
    id: str | int
```

## MCP Protocol (JSON-RPC 2.0)

Error codes:
```python
class MCPErrorCode(IntEnum):
    PARSE_ERROR = -32700
    INVALID_REQUEST = -32600
    METHOD_NOT_FOUND = -32601
    INVALID_PARAMS = -32602
    INTERNAL_ERROR = -32603
    RESOURCE_NOT_FOUND = -32000
    TOOL_EXECUTION_ERROR = -32001
    UNAUTHORIZED = -32003
    RATE_LIMIT_EXCEEDED = -32004
```

Transport interface:
```python
class MCPTransport(ABC):
    @abstractmethod
    async def send_message(self, msg: MCPMessage) -> None: ...
    @abstractmethod
    async def receive_message(self) -> MCPMessage: ...
    @abstractmethod
    async def close(self) -> None: ...
```

HTTP endpoints: `POST /mcp/v1/message`, `GET /mcp/v1/sse`

## MongoDB Setup

Motor connection:
```python
from motor.motor_asyncio import AsyncIOMotorClient

client = AsyncIOMotorClient(
    settings.MONGODB_URL,
    maxPoolSize=50,
    minPoolSize=10,
    serverSelectionTimeoutMS=5000
)
db = client.mcp_server

# Create indexes
await db.resources.create_index("uri", unique=True)
await db.resources.create_index([("name", "text"), ("description", "text")])
await db.resources.create_index("owner_id")
```

Repository pattern:
```python
class BaseRepository:
    def __init__(self, collection: AsyncIOMotorCollection):
        self.collection = collection
    
    async def find_by_id(self, id: str) -> dict | None:
        return await self.collection.find_one({"_id": ObjectId(id)})
    
    async def create(self, data: dict[str, Any]) -> str:
        data["created_at"] = datetime.utcnow()
        data["updated_at"] = datetime.utcnow()
        result = await self.collection.insert_one(data)
        return str(result.inserted_id)
```

## External JWT Auth

Verify JWT from external system:
```python
from fastapi import Depends, Security
from fastapi.security import HTTPBearer

security = HTTPBearer()

async def verify_external_jwt(
    credentials: HTTPAuthorizationCredentials = Security(security)
) -> dict[str, Any]:
    """Verify JWT from external auth system."""
    payload = jwt.decode(
        credentials.credentials,
        settings.JWT_PUBLIC_KEY,
        algorithms=["RS256"]
    )
    return payload  # Contains: user_id, email, roles, permissions

def require_permission(permission: str):
    async def check(user: dict = Depends(verify_external_jwt)) -> dict:
        if permission not in user.get("permissions", []):
            raise HTTPException(403, "Insufficient permissions")
        return user
    return check

# Usage
@app.post("/resources", dependencies=[Depends(require_permission("resources:write"))])
async def create_resource(resource: Resource, user: dict = Depends(verify_external_jwt)):
    return await repo.create({**resource.model_dump(), "owner_id": user["user_id"]})
```

Rate limiting:
```python
from slowapi import Limiter

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter

@app.post("/mcp/v1/message")
@limiter.limit("100/minute")
async def handle_message(request: Request): ...
```

## Observability

Structured logging:
```python
import structlog

logger = structlog.get_logger()
logger.info("mcp_request", method="tools/call", request_id="req-123", user_id="user-456")
```

OpenTelemetry:
```python
@tracer.start_as_current_span("execute_tool")
async def execute_tool(name: str, args: dict) -> dict:
    span = trace.get_current_span()
    span.set_attribute("tool.name", name)
    return await tool_executor.execute(name, args)
```

Prometheus metrics:
```python
from prometheus_client import Counter, Histogram

requests_total = Counter("mcp_requests_total", "Total requests", ["method", "status"])
request_duration = Histogram("mcp_request_duration_seconds", "Duration", ["method"])
```

## Testing

pytest with MongoDB:
```python
@pytest.fixture
async def test_db():
    client = AsyncIOMotorClient("mongodb://localhost:27017")
    db = client.mcp_test
    yield db
    await client.drop_database("mcp_test")

@pytest.mark.asyncio
async def test_create_resource(test_db):
    repo = ResourceRepository(test_db.resources)
    id = await repo.create({"uri": "file:///test.txt", "name": "Test"})
    resource = await repo.find_by_id(id)
    assert resource["uri"] == "file:///test.txt"
```

## Code Organization

```
services/protocol-handler/
├── src/
│   ├── main.py              # FastAPI entry
│   ├── config.py            # Settings
│   ├── database.py          # MongoDB connection
│   ├── api/v1/              # Endpoints
│   ├── core/                # Business logic
│   ├── models/              # Pydantic models
│   ├── repository/          # MongoDB repositories
│   └── utils/               # Logger, metrics
├── tests/
├── pyproject.toml
├── Dockerfile
└── README.md
```

Dependency injection:
```python
async def get_db() -> AsyncIOMotorDatabase:
    return db

async def get_repo(db: AsyncIOMotorDatabase = Depends(get_db)) -> ResourceRepository:
    return ResourceRepository(db.resources)

@app.get("/resources/{id}")
async def get_resource(id: str, repo: ResourceRepository = Depends(get_repo)):
    return await repo.find_by_id(id)
```

## MongoDB Collections

Resources:
```python
{
    "_id": ObjectId("..."),
    "uri": "file:///doc.pdf",
    "name": "Document",
    "mimeType": "application/pdf",
    "metadata": {"custom": "fields"},
    "owner_id": "user-123",
    "created_at": ISODate("..."),
    "updated_at": ISODate("...")
}
```

Prompts:
```python
{
    "_id": ObjectId("..."),
    "name": "code_review",
    "template": "Review: {{code}}",
    "variables": ["code", "language"],
    "version": 2,
    "versions": [{"version": 1, "template": "...", "created_at": ISODate("...")}]
}
```

## Error Handling

```python
class MCPException(Exception):
    def __init__(self, code: int, message: str, data: Any = None):
        self.code, self.message, self.data = code, message, data
    
    def to_json_rpc_error(self) -> dict:
        return {"code": self.code, "message": self.message, "data": self.data}

@app.exception_handler(MCPException)
async def mcp_exception_handler(request: Request, exc: MCPException):
    return JSONResponse(status_code=200, content={
        "jsonrpc": "2.0",
        "error": exc.to_json_rpc_error(),
        "id": (request.body as dict).get("id")
    })
```

## Health Checks

```python
@app.get("/health")
async def health(): return {"status": "healthy"}

@app.get("/ready")
async def ready():
    await db.command("ping")
    await redis.ping()
    return {"status": "ready", "checks": {"mongodb": True, "redis": True}}
```

## Deployment

Docker:
```dockerfile
FROM python:3.11-slim
RUN pip install fastapi motor pymongo uvicorn pydantic
COPY src/ ./src/
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Dependencies:
```toml
[tool.poetry.dependencies]
python = "^3.11"
fastapi = "^0.109.0"
uvicorn = {extras = ["standard"], version = "^0.27.0"}
motor = "^3.3.2"
pydantic = "^2.5.0"
python-jose = {extras = ["cryptography"], version = "^3.3.0"}
structlog = "^24.1.0"
opentelemetry-api = "^1.22.0"
prometheus-client = "^0.19.0"
```

## Checklist

- ✅ Python 3.11+ type hints
- ✅ Async I/O everywhere
- ✅ Pydantic validation
- ✅ MongoDB + Motor
- ✅ External JWT verification
- ✅ Structured logging
- ✅ OpenTelemetry tracing
- ✅ Prometheus metrics
- ✅ Rate limiting
- ✅ 80%+ test coverage
- ✅ Health/ready endpoints
- ✅ MongoDB indexes
