---
description: "MCP TypeScript client: async transports, reconnection, React hooks"
alwaysApply: false
globs: ["**/client/**/*.{ts,tsx}", "**/mcp-client/**/*.{ts,tsx}", "**/clients/**/*.{ts,tsx}"]
---

# MCP Client - TypeScript

TypeScript MCP client for Node.js and browser with stdio, HTTP+SSE, WebSocket.

## Quick Reference

**Features:** Promise-based, auto-reconnection, type-safe, event emitters, browser/Node compatible
**Transports:** StdioTransport (Node), HTTPSSETransport, WebSocketTransport
**Key:** React hooks included

## Core Standards

Strict types:
```typescript
async function callTool(
  name: string,
  arguments: Record<string, unknown>,
  timeout: number = 30000
): Promise<Record<string, unknown>> {
  // ...
}
```

Interfaces:
```typescript
interface MCPRequest {
  jsonrpc: "2.0";
  method: string;
  params?: Record<string, unknown>;
  id: string | number;
}

interface MCPResponse {
  jsonrpc: "2.0";
  result?: unknown;
  error?: MCPError;
  id: string | number;
}
```

## Transport Interface

```typescript
export interface MCPTransport {
  connect(): Promise<void>;
  send(message: MCPRequest): Promise<void>;
  receive(): Promise<MCPResponse>;
  close(): Promise<void>;
  isConnected: boolean;
  on(event: 'message', listener: (message: MCPResponse) => void): void;
  on(event: 'error', listener: (error: Error) => void): void;
  on(event: 'close', listener: () => void): void;
}
```

## Stdio Transport (Node.js)

```typescript
import { spawn } from 'child_process';
import { EventEmitter } from 'events';
import * as readline from 'readline';

export class StdioTransport extends EventEmitter implements MCPTransport {
  private process: ChildProcess | null = null;
  private rl: readline.Interface | null = null;

  constructor(private command: string, private args: string[] = []) {
    super();
  }

  async connect(): Promise<void> {
    this.process = spawn(this.command, this.args, { stdio: ['pipe', 'pipe', 'pipe'] });
    this.rl = readline.createInterface({ input: this.process.stdout! });
    
    this.rl.on('line', (line) => {
      this.emit('message', JSON.parse(line));
    });
    
    this.process.on('close', () => this.emit('close'));
  }

  async send(message: MCPRequest): Promise<void> {
    this.process!.stdin!.write(JSON.stringify(message) + '\n');
  }
}
```

## HTTP+SSE Transport

```typescript
export class HTTPSSETransport extends EventEmitter implements MCPTransport {
  private eventSource: EventSource | null = null;

  constructor(private baseUrl: string, private headers: Record<string, string> = {}) {
    super();
  }

  async connect(): Promise<void> {
    this.eventSource = new EventSource(`${this.baseUrl}/mcp/v1/sse`);
    
    this.eventSource.onmessage = (event) => {
      this.emit('message', JSON.parse(event.data));
    };
  }

  async send(message: MCPRequest): Promise<void> {
    await fetch(`${this.baseUrl}/mcp/v1/message`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', ...this.headers },
      body: JSON.stringify(message)
    });
  }
}
```

## WebSocket Transport

```typescript
export class WebSocketTransport extends EventEmitter implements MCPTransport {
  private ws: WebSocket | null = null;

  constructor(private url: string) {
    super();
  }

  async connect(): Promise<void> {
    return new Promise((resolve) => {
      this.ws = new WebSocket(this.url);
      this.ws.onopen = () => resolve();
      this.ws.onmessage = (event) => this.emit('message', JSON.parse(event.data));
      this.ws.onclose = () => this.emit('close');
    });
  }

  async send(message: MCPRequest): Promise<void> {
    this.ws!.send(JSON.stringify(message));
  }
}
```

## MCP Client

```typescript
import { EventEmitter } from 'events';
import { v4 as uuidv4 } from 'uuid';

export class MCPClient extends EventEmitter {
  private pendingRequests = new Map<string, {
    resolve: (value: unknown) => void;
    reject: (reason: Error) => void;
    timeout: NodeJS.Timeout;
  }>();

  constructor(
    private transport: MCPTransport,
    private options: { autoReconnect?: boolean; maxRetries?: number; timeout?: number } = {}
  ) {
    super();
    this.transport.on('message', this.handleMessage.bind(this));
    this.transport.on('close', this.handleClose.bind(this));
  }

  async connect(): Promise<void> {
    await this.transport.connect();
    this.emit('connected');
  }

  private handleMessage(message: MCPResponse): void {
    if (message.id !== undefined) {
      const pending = this.pendingRequests.get(String(message.id));
      if (pending) {
        clearTimeout(pending.timeout);
        this.pendingRequests.delete(String(message.id));
        message.error ? pending.reject(new Error(message.error.message)) : pending.resolve(message.result);
      }
    }
  }

  private async handleClose(): Promise<void> {
    if (this.options.autoReconnect) {
      await this.reconnect();
    }
  }

  private async reconnect(): Promise<void> {
    const maxRetries = this.options.maxRetries ?? 3;
    for (let i = 0; i < maxRetries; i++) {
      await new Promise(r => setTimeout(r, Math.pow(2, i) * 1000));
      try {
        await this.transport.connect();
        return;
      } catch {}
    }
  }

  async request(method: string, params?: Record<string, unknown>): Promise<unknown> {
    const id = uuidv4();
    const timeout = this.options.timeout ?? 30000;

    return new Promise((resolve, reject) => {
      const timer = setTimeout(() => {
        this.pendingRequests.delete(id);
        reject(new Error('Timeout'));
      }, timeout);

      this.pendingRequests.set(id, { resolve, reject, timeout: timer });
      this.transport.send({ jsonrpc: "2.0", method, params, id });
    });
  }

  async callTool(name: string, arguments_: Record<string, unknown>): Promise<unknown> {
    return this.request('tools/call', { name, arguments: arguments_ });
  }

  async listResources(): Promise<unknown[]> {
    const result = await this.request('resources/list') as { resources: unknown[] };
    return result.resources;
  }
}
```

## React Hook

```typescript
import { useEffect, useState, useCallback } from 'react';

export function useMCPClient(transport: MCPTransport) {
  const [client, setClient] = useState<MCPClient | null>(null);
  const [connected, setConnected] = useState(false);

  useEffect(() => {
    const mcpClient = new MCPClient(transport);
    mcpClient.on('connected', () => setConnected(true));
    mcpClient.on('disconnected', () => setConnected(false));
    mcpClient.connect();
    setClient(mcpClient);

    return () => { mcpClient.close(); };
  }, [transport]);

  const callTool = useCallback(async (name: string, args: Record<string, unknown>) => {
    return client?.callTool(name, args);
  }, [client]);

  return { client, connected, callTool };
}

// Usage
function MyComponent() {
  const { connected, callTool } = useMCPClient(
    new HTTPSSETransport('https://api.example.com')
  );

  const handleClick = async () => {
    const result = await callTool('calculator', { a: 1, b: 2 });
  };

  return connected ? <button onClick={handleClick}>Call</button> : <div>Connecting...</div>;
}
```

## Usage

Stdio (Node.js):
```typescript
const client = new MCPClient(new StdioTransport('python', ['server.py']));
await client.connect();
const result = await client.callTool('calculator', { a: 1, b: 2 });
```

HTTP+SSE:
```typescript
const transport = new HTTPSSETransport('https://api.example.com', {
  Authorization: 'Bearer token'
});
const client = new MCPClient(transport, { autoReconnect: true });
await client.connect();
```

WebSocket:
```typescript
const client = new MCPClient(new WebSocketTransport('wss://api.example.com/mcp/v1/ws'));
await client.connect();
```

## Testing

Mock transport:
```typescript
export class MockTransport extends EventEmitter implements MCPTransport {
  public sentMessages: MCPRequest[] = [];

  async send(message: MCPRequest): Promise<void> {
    this.sentMessages.push(message);
  }

  mockResponse(response: MCPResponse): void {
    this.emit('message', response);
  }
}

it('should call tool', async () => {
  const transport = new MockTransport();
  const client = new MCPClient(transport);
  await client.connect();

  const promise = client.callTool('calculator', { a: 1, b: 2 });
  transport.mockResponse({ jsonrpc: "2.0", result: { output: 3 }, id: transport.sentMessages[0].id });

  expect(await promise).toEqual({ output: 3 });
});
```

## Dependencies

```json
{
  "dependencies": {
    "uuid": "^9.0.0",
    "ws": "^8.14.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@types/uuid": "^9.0.0",
    "typescript": "^5.3.0",
    "jest": "^29.7.0"
  }
}
```

## Checklist

- ✅ Strict TypeScript
- ✅ Promise-based async
- ✅ Stdio, HTTP+SSE, WebSocket
- ✅ Auto-reconnection
- ✅ Timeout handling
- ✅ Event emitters
- ✅ Browser + Node.js
- ✅ React hooks
- ✅ Mock transport
