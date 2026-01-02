---
description: "MCP TypeScript server: Fastify + MongoDB + JWKS + OpenTelemetry"
alwaysApply: false
globs: ["**/services/**/*.ts", "**/apps/server/**/*.ts", "**/packages/server/**/*.ts"]
---

# MCP Server - TypeScript

Production MCP server: TypeScript/Node 20+, Fastify, MongoDB, Redis. External auth with JWKS.

## Quick Reference

**Stack:** Fastify, Mongoose, Redis, Zod, Pino, OpenTelemetry, Prometheus
**Auth:** External JWT (JWKS) - no user DB
**DB:** MongoDB for resources/prompts/tools/logs
**Transports:** stdio, HTTP+SSE, WebSocket

## Core Standards

Strict types:
```typescript
async function process(
  data: Record<string, unknown>,
  callback?: (result: string) => void
): Promise<[number, string]> {
  // ...
}
```

Zod validation:
```typescript
import { z } from 'zod';

const MCPRequestSchema = z.object({
  jsonrpc: z.literal('2.0'),
  method: z.string().min(1).max(100),
  params: z.record(z.unknown()).optional().default({}),
  id: z.union([z.string(), z.number()])
});

type MCPRequest = z.infer<typeof MCPRequestSchema>;
```

## MCP Protocol

Error codes:
```typescript
export enum MCPErrorCode {
  ParseError = -32700,
  InvalidRequest = -32600,
  MethodNotFound = -32601,
  InvalidParams = -32602,
  InternalError = -32603,
  ResourceNotFound = -32000,
  ToolExecutionError = -32001,
  Unauthorized = -32003,
  RateLimitExceeded = -32004
}
```

Transport interface:
```typescript
export interface MCPTransport {
  send(message: MCPMessage): Promise<void>;
  receive(): Promise<MCPMessage>;
  close(): Promise<void>;
  readonly isConnected: boolean;
}
```

## Fastify Setup

```typescript
import Fastify from 'fastify';
import cors from '@fastify/cors';
import helmet from '@fastify/helmet';
import rateLimit from '@fastify/rate-limit';

const server = Fastify({ logger: true });

await server.register(helmet);
await server.register(cors);
await server.register(rateLimit, { max: 100, timeWindow: '1 minute' });

server.get('/health', async () => ({ status: 'healthy' }));

server.get('/ready', async () => {
  await mongoose.connection.db.admin().ping();
  await redis.ping();
  return { status: 'ready' };
});

server.post('/mcp/v1/message', async (request) => {
  const mcpRequest = MCPRequestSchema.parse(request.body);
  return await handleMCPRequest(mcpRequest, request.user);
});
```

## MongoDB + Mongoose

Schema:
```typescript
import mongoose, { Schema, Document } from 'mongoose';

export interface IResource extends Document {
  uri: string;
  name: string;
  mimeType: string;
  metadata: Record<string, unknown>;
  ownerId: string;
}

const ResourceSchema = new Schema<IResource>({
  uri: { type: String, required: true, unique: true, index: true },
  name: { type: String, required: true, index: 'text' },
  mimeType: { type: String, required: true },
  metadata: { type: Schema.Types.Mixed, default: {} },
  ownerId: { type: String, required: true, index: true }
}, { timestamps: true });

ResourceSchema.index({ name: 'text', description: 'text' });

export const Resource = mongoose.model<IResource>('Resource', ResourceSchema);
```

Repository:
```typescript
export class BaseRepository<T extends Document> {
  constructor(protected model: Model<T>) {}

  async findById(id: string): Promise<T | null> {
    return this.model.findById(id);
  }

  async create(data: Partial<T>): Promise<T> {
    return new this.model(data).save();
  }
}

export class ResourceRepository extends BaseRepository<IResource> {
  constructor() { super(Resource); }

  async findByUri(uri: string): Promise<IResource | null> {
    return this.model.findOne({ uri });
  }

  async search(query: string): Promise<IResource[]> {
    return this.model.find({ $text: { $search: query } }).limit(100);
  }
}
```

## JWKS JWT Auth

```typescript
import jwt from 'jsonwebtoken';
import jwksClient from 'jwks-rsa';

const client = jwksClient({
  jwksUri: process.env.JWKS_URI!,
  cache: true,
  cacheMaxAge: 86400000
});

function getKey(header: jwt.JwtHeader, callback: jwt.SigningKeyCallback) {
  client.getSigningKey(header.kid, (err, key) => {
    const signingKey = key?.getPublicKey();
    callback(err, signingKey);
  });
}

export interface JWTPayload {
  user_id: string;
  permissions: string[];
}

export async function authPlugin(fastify: FastifyInstance) {
  fastify.addHook('preHandler', async (request, reply) => {
    const token = request.headers.authorization?.substring(7);
    if (!token) throw new Error('Missing token');

    request.user = await new Promise<JWTPayload>((resolve, reject) => {
      jwt.verify(token, getKey, { algorithms: ['RS256'] }, (err, decoded) => {
        err ? reject(err) : resolve(decoded as JWTPayload);
      });
    });
  });
}

export function requirePermission(permission: string) {
  return async (request: FastifyRequest) => {
    if (!request.user?.permissions.includes(permission)) {
      throw new Error('Insufficient permissions');
    }
  };
}
```

## HTTP+SSE Transport

```typescript
class SSEConnection {
  constructor(private reply: FastifyReply) {}
  
  send(message: MCPMessage) {
    this.reply.raw.write(`data: ${JSON.stringify(message)}\n\n`);
  }
}

const connections = new Map<string, SSEConnection>();

server.get('/mcp/v1/sse', async (request, reply) => {
  reply.raw.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Connection': 'keep-alive'
  });

  const connection = new SSEConnection(reply);
  connections.set(request.user!.user_id, connection);

  request.raw.on('close', () => connections.delete(request.user!.user_id));
});
```

## WebSocket

```typescript
import fastifyWebsocket from '@fastify/websocket';

await server.register(fastifyWebsocket);

server.register(async (fastify) => {
  fastify.get('/mcp/v1/ws', { websocket: true }, (connection, request) => {
    connection.socket.on('message', async (data) => {
      const message = JSON.parse(data.toString());
      const response = await handleMCPRequest(message, request.user);
      connection.socket.send(JSON.stringify(response));
    });
  });
});
```

## Observability

Pino logging:
```typescript
server.addHook('onRequest', async (request) => {
  request.log.info({
    method: request.method,
    url: request.url,
    requestId: request.id
  });
});
```

OpenTelemetry:
```typescript
import { trace } from '@opentelemetry/api';

const tracer = trace.getTracer('mcp-server');

async function executeTool(name: string, args: Record<string, unknown>) {
  return tracer.startActiveSpan('execute_tool', async (span) => {
    span.setAttribute('tool.name', name);
    const result = await toolExecutor.execute(name, args);
    span.end();
    return result;
  });
}
```

Prometheus:
```typescript
import promClient from 'prom-client';

const register = new promClient.Registry();
const requestCounter = new promClient.Counter({
  name: 'mcp_requests_total',
  help: 'Total requests',
  labelNames: ['method', 'status'],
  registers: [register]
});

server.get('/metrics', async () => register.metrics());
```

## Error Handling

```typescript
export class MCPException extends Error {
  constructor(
    public code: MCPErrorCode,
    message: string,
    public data?: unknown
  ) {
    super(message);
  }

  toJSONRPC(): MCPError {
    return { code: this.code, message: this.message, data: this.data };
  }
}

server.setErrorHandler((error, request, reply) => {
  if (error instanceof MCPException) {
    return reply.code(200).send({
      jsonrpc: '2.0',
      error: error.toJSONRPC(),
      id: (request.body as any)?.id
    });
  }
  return reply.code(500).send({ error: 'Internal server error' });
});
```

## Testing

Jest with MongoDB memory server:
```typescript
import { MongoMemoryServer } from 'mongodb-memory-server';
import mongoose from 'mongoose';

let mongoServer: MongoMemoryServer;

beforeAll(async () => {
  mongoServer = await MongoMemoryServer.create();
  await mongoose.connect(mongoServer.getUri());
});

afterAll(async () => {
  await mongoose.disconnect();
  await mongoServer.stop();
});

it('should create resource', async () => {
  const repo = new ResourceRepository();
  const resource = await repo.create({
    uri: 'file:///test.txt',
    name: 'Test',
    ownerId: 'user-123'
  });
  expect(resource.uri).toBe('file:///test.txt');
});
```

## Code Organization

```
services/protocol-handler/
├── src/
│   ├── index.ts             # Entry
│   ├── server.ts            # Fastify
│   ├── routes/              # Endpoints
│   ├── services/            # Business logic
│   ├── repositories/        # Data access
│   ├── models/              # Mongoose
│   ├── middleware/          # Auth
│   └── types/               # Types
├── tests/
├── package.json
└── tsconfig.json
```

## Deployment

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY src ./src
RUN npm run build

FROM node:20-alpine
WORKDIR /app
RUN adduser -D -u 1000 appuser
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist
USER appuser
CMD ["node", "dist/index.js"]
```

Dependencies:
```json
{
  "dependencies": {
    "fastify": "^4.25.0",
    "@fastify/cors": "^8.4.0",
    "@fastify/helmet": "^11.1.1",
    "@fastify/websocket": "^8.3.1",
    "mongoose": "^8.0.0",
    "jsonwebtoken": "^9.0.2",
    "jwks-rsa": "^3.1.0",
    "zod": "^3.22.4",
    "pino": "^8.16.0"
  }
}
```

## Checklist

- ✅ Strict TypeScript
- ✅ Zod validation
- ✅ Fastify
- ✅ Mongoose
- ✅ JWKS JWT
- ✅ HTTP+SSE, WebSocket
- ✅ Pino logging
- ✅ OpenTelemetry
- ✅ Prometheus
- ✅ Jest testing
