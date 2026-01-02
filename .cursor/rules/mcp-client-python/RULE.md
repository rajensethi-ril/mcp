---
description: "MCP Python client: async transports, reconnection, event-driven"
alwaysApply: false
globs: ["**/client/**/*.py", "**/mcp_client/**/*.py", "**/clients/**/*.py"]
---

# MCP Client - Python

Python 3.11+ MCP client with stdio, HTTP+SSE, WebSocket transports.

## Quick Reference

**Features:** Async/await, auto-reconnection, request/response correlation, event-driven, type-safe
**Transports:** StdioTransport, HTTPSSETransport, WebSocketTransport
**Key:** Context manager support (`async with`)

## Core Standards

Type hints always:
```python
async def call_tool(
    name: str,
    arguments: dict[str, Any],
    timeout: float = 30.0
) -> dict[str, Any]:
    ...
```

Async I/O:
```python
# ✅ Good
async with MCPClient(HTTPSSETransport("https://api.example.com")) as client:
    result = await client.call_tool("calculator", {"a": 1, "b": 2})
```

## Transport Interface

```python
from abc import ABC, abstractmethod

class MCPTransport(ABC):
    @abstractmethod
    async def connect(self) -> None: ...
    
    @abstractmethod
    async def send(self, message: dict[str, Any]) -> None: ...
    
    @abstractmethod
    async def receive(self) -> dict[str, Any]: ...
    
    @abstractmethod
    async def close(self) -> None: ...
    
    @property
    @abstractmethod
    def is_connected(self) -> bool: ...
```

## Stdio Transport

```python
class StdioTransport(MCPTransport):
    def __init__(self, command: list[str]):
        self.command = command
        self.process: asyncio.subprocess.Process | None = None
    
    async def connect(self) -> None:
        self.process = await asyncio.create_subprocess_exec(
            *self.command,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE
        )
    
    async def send(self, message: dict[str, Any]) -> None:
        data = json.dumps(message).encode() + b"\n"
        self.process.stdin.write(data)
        await self.process.stdin.drain()
    
    async def receive(self) -> dict[str, Any]:
        line = await self.process.stdout.readline()
        return json.loads(line.decode())
```

## HTTP+SSE Transport

```python
import aiohttp

class HTTPSSETransport(MCPTransport):
    def __init__(self, base_url: str, headers: dict[str, str] | None = None):
        self.base_url = base_url.rstrip("/")
        self.headers = headers or {}
        self.session: aiohttp.ClientSession | None = None
        self.message_queue: asyncio.Queue = asyncio.Queue()
    
    async def connect(self) -> None:
        self.session = aiohttp.ClientSession(headers=self.headers)
        asyncio.create_task(self._listen_sse())
    
    async def _listen_sse(self) -> None:
        async with self.session.get(f"{self.base_url}/mcp/v1/sse") as response:
            async for line in response.content:
                if line.startswith(b"data: "):
                    data = json.loads(line[6:].decode())
                    await self.message_queue.put(data)
    
    async def send(self, message: dict[str, Any]) -> None:
        async with self.session.post(f"{self.base_url}/mcp/v1/message", json=message) as resp:
            resp.raise_for_status()
    
    async def receive(self) -> dict[str, Any]:
        return await self.message_queue.get()
```

## WebSocket Transport

```python
import websockets

class WebSocketTransport(MCPTransport):
    def __init__(self, url: str, headers: dict[str, str] | None = None):
        self.url = url
        self.headers = headers
        self.ws: websockets.WebSocketClientProtocol | None = None
    
    async def connect(self) -> None:
        self.ws = await websockets.connect(self.url, extra_headers=self.headers)
    
    async def send(self, message: dict[str, Any]) -> None:
        await self.ws.send(json.dumps(message))
    
    async def receive(self) -> dict[str, Any]:
        data = await self.ws.recv()
        return json.loads(data)
```

## MCP Client

```python
from uuid import uuid4

class MCPClient:
    def __init__(
        self,
        transport: MCPTransport,
        auto_reconnect: bool = True,
        max_retries: int = 3
    ):
        self.transport = transport
        self.auto_reconnect = auto_reconnect
        self.max_retries = max_retries
        self.pending_requests: dict[str, asyncio.Future] = {}
        self.event_handlers: dict[str, list[Callable]] = {}
    
    async def __aenter__(self):
        await self.connect()
        return self
    
    async def __aexit__(self, *args):
        await self.close()
    
    async def connect(self) -> None:
        await self.transport.connect()
        asyncio.create_task(self._receive_loop())
    
    async def _receive_loop(self) -> None:
        while self.transport.is_connected:
            try:
                message = await self.transport.receive()
                await self._handle_message(message)
            except Exception:
                if self.auto_reconnect:
                    await self._reconnect()
    
    async def _reconnect(self) -> None:
        for attempt in range(self.max_retries):
            await asyncio.sleep(2 ** attempt)  # Exponential backoff
            try:
                await self.transport.connect()
                return
            except Exception:
                if attempt == self.max_retries - 1:
                    raise
    
    async def request(
        self,
        method: str,
        params: dict[str, Any] | None = None,
        timeout: float = 30.0
    ) -> Any:
        request_id = str(uuid4())
        message = {
            "jsonrpc": "2.0",
            "method": method,
            "params": params or {},
            "id": request_id
        }
        
        future = asyncio.Future()
        self.pending_requests[request_id] = future
        await self.transport.send(message)
        
        try:
            return await asyncio.wait_for(future, timeout=timeout)
        except asyncio.TimeoutError:
            self.pending_requests.pop(request_id, None)
            raise TimeoutError(f"Request {method} timed out")
    
    async def call_tool(self, name: str, arguments: dict[str, Any]) -> dict[str, Any]:
        return await self.request("tools/call", {"name": name, "arguments": arguments})
    
    async def list_resources(self) -> list[dict[str, Any]]:
        result = await self.request("resources/list")
        return result.get("resources", [])
    
    def on(self, event: str, handler: Callable) -> None:
        """Register event handler for notifications."""
        if event not in self.event_handlers:
            self.event_handlers[event] = []
        self.event_handlers[event].append(handler)
```

## Usage

Stdio:
```python
async with MCPClient(StdioTransport(["python", "server.py"])) as client:
    result = await client.call_tool("calculator", {"a": 1, "b": 2})
```

HTTP+SSE with auth:
```python
transport = HTTPSSETransport(
    "https://api.example.com",
    headers={"Authorization": "Bearer token"}
)
async with MCPClient(transport) as client:
    resources = await client.list_resources()
```

WebSocket:
```python
transport = WebSocketTransport("wss://api.example.com/mcp/v1/ws")
async with MCPClient(transport) as client:
    result = await client.call_tool("search", {"query": "hello"})
    client.on("resource_updated", lambda params: print(params))
```

## Testing

Mock transport:
```python
class MockTransport(MCPTransport):
    def __init__(self):
        self.sent_messages: list[dict] = []
        self.responses: asyncio.Queue = asyncio.Queue()
    
    async def send(self, message: dict[str, Any]) -> None:
        self.sent_messages.append(message)
    
    def mock_response(self, response: dict[str, Any]) -> None:
        self.responses.put_nowait(response)

@pytest.mark.asyncio
async def test_call_tool():
    transport = MockTransport()
    client = MCPClient(transport)
    await client.connect()
    
    transport.mock_response({
        "jsonrpc": "2.0",
        "result": {"output": 3},
        "id": transport.sent_messages[0]["id"]
    })
    
    result = await client.call_tool("calculator", {"a": 1, "b": 2})
    assert result["output"] == 3
```

## Dependencies

```toml
[tool.poetry.dependencies]
python = "^3.11"
aiohttp = "^3.9.0"
websockets = "^12.0"
pydantic = "^2.5.0"

[tool.poetry.dev-dependencies]
pytest = "^7.4.0"
pytest-asyncio = "^0.21.0"
pytest-cov = "^4.1.0"
```

## Checklist

- ✅ Type hints everywhere
- ✅ Async/await for I/O
- ✅ Stdio, HTTP+SSE, WebSocket
- ✅ Auto-reconnection
- ✅ Request/response correlation
- ✅ Timeout handling
- ✅ Event-driven notifications
- ✅ Context manager support
- ✅ Mock transport for testing
