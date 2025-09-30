# Hono + OpenAPI Project Architecture Guide

## Overview
This guide documents the organization pattern for building REST APIs with Hono framework and OpenAPI/Swagger integration on Cloudflare Workers.

## Project Structure

```
src/
├── index.ts                 # Application entry point
├── lib/
│   └── openapi.ts          # OpenAPI app configuration
├── swagger/
│   └── index.ts            # Swagger UI setup
├── routes/                 # Route definitions (OpenAPI specs)
│   ├── billing-routes.ts
│   ├── client-routes.ts
│   └── template-routes.ts
├── handlers/               # Request handlers (business logic)
│   ├── billing-handlers.ts
│   ├── client-handlers.ts
│   └── template-handlers.ts
├── schemas/                # Zod validation schemas
│   └── index.ts
├── services/              # Business services
│   ├── pdf.service.ts
│   ├── storage.service.ts
│   └── template-processor.ts
└── db/                    # Database schemas and migrations
    └── migrations/
        ├── schema.ts
        └── relations.ts
```

## Core Components

### 1. OpenAPI App Creation (`lib/openapi.ts`)

```typescript
import { OpenAPIHono } from '@hono/zod-openapi';

// Environment type for better type safety
export type Env = {
    Bindings: {
        // Define your Cloudflare bindings here
        NEON_DB?: string;
        JWT_SECRET?: string;
        SWAGGER_USERNAME?: string;
        SWAGGER_PASSWORD?: string;
        ENVIRONMENT?: string;
        PDF_GENERATOR: DurableObjectNamespace;
    }
};

// Configure OpenAPI app following modern Hono patterns
export function createOpenAPIApp() {
    const app = new OpenAPIHono<Env>();
    return app;
}
```

**Purpose**: Creates typed OpenAPI-enabled Hono app with environment bindings.

**Key Points**:
- Define all Cloudflare bindings (D1, R2, KV, Durable Objects, etc.) in the `Env` type
- Export the `Env` type for use throughout the application
- Keep app creation minimal - middleware is added in `index.ts`

### 2. Main Application Setup (`index.ts`)

```typescript
import { createOpenAPIApp } from "./lib/openapi";
import { logger } from "hono/logger";
import { prettyJSON } from "hono/pretty-json";
import { cors } from "hono/cors";
import { setupSwagger } from "./swagger";

// Import route definitions
import {
  createBillingRoute,
  generatePDFRoute,
  getBillingRoute,
  updateBillingRoute
} from "./routes/billing-routes";

// Import handlers
import {
  createBillingHandler,
  generatePDFHandler,
  getBillingHandler,
  updateBillingHandler
} from "./handlers/billing-handlers";

const app = createOpenAPIApp();

// Global middleware
app.use('*', logger());
app.use('*', prettyJSON());

// CORS configuration
app.use('*', cors({
  origin: ['http://localhost:8082', 'https://yourdomain.com'],
  allowMethods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'],
  allowHeaders: ['Content-Type', 'Authorization'],
  credentials: true
}));

// Setup Swagger documentation
setupSwagger(app);

// Register routes with handlers
app.openapi(createBillingRoute, createBillingHandler);
app.openapi(generatePDFRoute, generatePDFHandler);
app.openapi(getBillingRoute, getBillingHandler);
app.openapi(updateBillingRoute, updateBillingHandler);

export default app;
```

**Key Pattern**:
- Apply global middleware first
- Setup Swagger before route registration
- Separate route definitions from handlers
- Register using `app.openapi(route, handler)` pattern
- Export `app` as default for Cloudflare Workers

### 3. Swagger Configuration (`swagger/index.ts`)

```typescript
import { OpenAPIHono } from '@hono/zod-openapi';
import { swaggerUI } from '@hono/swagger-ui';
import { basicAuth } from 'hono/basic-auth';
import type { Env } from '../lib/openapi';

export function setupSwagger(app: OpenAPIHono<Env>) {
    // Authentication middleware for Swagger UI
    app.use('/docs/*', async (c, next) => {
        const username = c.env?.SWAGGER_USERNAME || 'admin';
        const password = c.env?.SWAGGER_PASSWORD || 'default-password';

        return basicAuth({
            username,
            password,
            realm: 'API Documentation'
        })(c, next);
    });

    // Serve OpenAPI spec as JSON
    app.doc('/docs/openapi.json', {
        openapi: '3.0.0',
        info: {
            title: 'Your API',
            version: '1.0.0',
            description: `
# Your API Documentation

API REST completa para gestión de recursos, incluyendo:

- 📄 **Gestión de recursos**
- 👥 **Gestión de clientes**
- 📊 **Analytics**

## Autenticación

Todos los endpoints requieren autenticación JWT:

\`\`\`
Authorization: Bearer <tu-jwt-token>
\`\`\`

## Códigos de Error

| Código | Descripción |
|--------|-------------|
| 400 | Datos de entrada inválidos |
| 401 | No autenticado |
| 403 | Sin permisos |
| 404 | Recurso no encontrado |
| 500 | Error interno del servidor |
            `,
            contact: {
                name: 'Dev Team',
                email: 'dev@example.com',
                url: 'https://example.com'
            }
        }
    });

    // Serve Swagger UI
    app.get('/docs', swaggerUI({
        url: '/docs/openapi.json',
    }));

    // Health check endpoint
    app.get('/health', (c) => {
        return c.json({
            status: 'ok',
            timestamp: new Date().toISOString(),
            version: '1.0.0',
            environment: c.env?.ENVIRONMENT || 'development'
        });
    });

    // Root redirect to docs
    app.get('/', (c) => {
        return c.redirect('/docs');
    });

    return app;
}
```

**Features**:
- Basic auth protection for documentation
- OpenAPI spec at `/docs/openapi.json`
- Swagger UI at `/docs`
- Health check endpoint
- Root redirect to documentation

### 4. Route Definitions (`routes/*-routes.ts`)

Define OpenAPI routes using `createRoute` from `@hono/zod-openapi`:

```typescript
import { createRoute } from '@hono/zod-openapi';
import { z } from 'zod';
import { BillingSchema, CreateBillingSchema, ErrorSchema } from '../schemas';

// POST /api/billings
export const createBillingRoute = createRoute({
  method: 'post',
  path: '/api/billings',
  tags: ['Billings'],
  summary: 'Create new billing',
  description: 'Create a new billing record',
  request: {
    body: {
      content: {
        'application/json': {
          schema: CreateBillingSchema
        }
      }
    }
  },
  responses: {
    201: {
      content: {
        'application/json': {
          schema: BillingSchema
        }
      },
      description: 'Billing created successfully'
    },
    400: {
      content: {
        'application/json': {
          schema: ErrorSchema
        }
      },
      description: 'Invalid input data'
    },
    404: {
      content: {
        'application/json': {
          schema: ErrorSchema
        }
      },
      description: 'Client not found'
    }
  }
});

// GET /api/billings/:id
export const getBillingRoute = createRoute({
  method: 'get',
  path: '/api/billings/{id}',
  tags: ['Billings'],
  summary: 'Get billing details',
  description: 'Retrieve details of a specific billing',
  request: {
    params: z.object({
      id: z.string().openapi({ example: 'billing-123' })
    })
  },
  responses: {
    200: {
      content: {
        'application/json': {
          schema: BillingSchema
        }
      },
      description: 'Billing details retrieved successfully'
    },
    404: {
      content: {
        'application/json': {
          schema: ErrorSchema
        }
      },
      description: 'Billing not found'
    }
  }
});

// POST /api/billings/:id/generate-pdf
export const generatePDFRoute = createRoute({
  method: 'post',
  path: '/api/billings/{id}/generate-pdf',
  tags: ['Billings'],
  summary: 'Generate PDF',
  description: 'Generate PDF document for a billing',
  request: {
    params: z.object({
      id: z.string().openapi({ example: 'billing-123' })
    })
  },
  responses: {
    200: {
      content: {
        'application/pdf': {
          schema: { type: 'string', format: 'binary' }
        }
      },
      description: 'PDF generated successfully'
    },
    404: {
      content: {
        'application/json': {
          schema: ErrorSchema
        }
      },
      description: 'Billing not found'
    },
    500: {
      content: {
        'application/json': {
          schema: ErrorSchema
        }
      },
      description: 'PDF generation failed'
    }
  }
});
```

**Key Points**:
- Use Zod schemas for validation
- Define all HTTP methods, paths, tags, and response types
- Include examples in schema definitions using `.openapi()`
- Schemas automatically generate OpenAPI documentation
- Group related routes in same file
- Export each route individually for selective registration

### 5. Handlers (`handlers/*-handlers.ts`)

Implement business logic separately:

```typescript
import type { Context } from 'hono';
import { neon } from "@neondatabase/serverless";
import { drizzle } from "drizzle-orm/neon-http";
import { eq } from "drizzle-orm";
import { billings, clients, billingItems } from "../db/migrations/schema";

export const createBillingHandler = async (c: Context) => {
    try {
        const { clientId, items, ...newBill } = await c.req.json();

        // Initialize database connection
        const sql = neon(c.env.NEON_DB);
        const db = drizzle(sql);

        // Validate client exists
        const client = await db
            .select()
            .from(clients)
            .where(eq(clients.id, clientId))
            .limit(1);

        if (!client || client.length === 0) {
            return c.json({ error: 'Client not found' }, 404);
        }

        // Create billing record
        const [billing] = await db
            .insert(billings)
            .values({
                clientId,
                billingCode: '',
                ...newBill,
                status: 'draft',
            })
            .returning();

        // Create billing items
        if (items && items.length > 0) {
            const billingItemsData = items.map((item: any) => ({
                billingId: billing.id,
                description: item.description,
                quantity: item.quantity,
                unitPrice: item.unitPrice,
                total: item.total || item.quantity * item.unitPrice
            }));

            await db.insert(billingItems).values(billingItemsData);
        }

        return c.json({
            success: true,
            billing
        }, 201);

    } catch (error: any) {
        console.error('Error creating billing:', error);
        return c.json({ error: 'Error creating billing' }, 500);
    }
};

export const getBillingHandler = async (c: Context) => {
    try {
        const id = c.req.param('id');

        const sql = neon(c.env.NEON_DB);
        const db = drizzle(sql);

        const billingResult = await db
            .select()
            .from(billings)
            .where(eq(billings.id, id))
            .limit(1);

        if (!billingResult || billingResult.length === 0) {
            return c.json({ error: 'Billing not found' }, 404);
        }

        return c.json(billingResult[0]);

    } catch (error: any) {
        console.error('Error getting billing:', error);
        return c.json({ error: 'Billing not found' }, 404);
    }
};

export const generatePDFHandler = async (c: Context) => {
    try {
        const id = c.req.param('id');

        const sql = neon(c.env.NEON_DB);
        const db = drizzle(sql);

        // Get billing with related data
        const billing = await db
            .select()
            .from(billings)
            .where(eq(billings.id, id))
            .limit(1);

        if (!billing || billing.length === 0) {
            return c.json({ error: 'Billing not found' }, 404);
        }

        // Generate PDF using Durable Object
        const containerId = c.env.PDF_GENERATOR.idFromName(`pdf-${id}`);
        const container = c.env.PDF_GENERATOR.get(containerId);

        if (!container) {
            return c.json({ error: 'PDF generation service unavailable' }, 503);
        }

        const pdfResponse = await container.fetch(
            'http://pdf-generator/generate-pdf',
            {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ billing: billing[0] })
            }
        );

        if (!pdfResponse.ok) {
            return c.json({ error: 'PDF generation failed' }, 500);
        }

        const pdfBuffer = new Uint8Array(await pdfResponse.arrayBuffer());

        c.header('Content-Type', 'application/pdf');
        c.header('Content-Disposition', `attachment; filename="invoice-${billing[0].billCode}.pdf"`);

        return c.body(pdfBuffer);

    } catch (error: any) {
        console.error('Error generating PDF:', error);
        return c.json({ error: 'PDF generation failed' }, 500);
    }
};
```

**Best Practices**:
- Access environment bindings via `c.env`
- Use proper HTTP status codes
- Always handle errors with try-catch
- Return consistent error format
- Log errors for debugging
- Validate input data
- Use transactions for multi-step operations

### 6. Schemas (`schemas/index.ts`)

Centralize Zod validation schemas:

```typescript
import { z } from 'zod';

// Error Schema
export const ErrorSchema = z.object({
  error: z.string(),
  code: z.string().optional()
});

// Billing Item Schema
export const BillingItemSchema = z.object({
  id: z.string().optional(),
  description: z.string(),
  quantity: z.number().positive(),
  unitPrice: z.number().positive(),
  total: z.number().positive()
});

// Create Billing Schema (input)
export const CreateBillingSchema = z.object({
  billCode: z.string(),
  billSeries: z.string(),
  billingType: z.number(),
  clientId: z.string().uuid(),
  clientName: z.string(),
  clientDocument: z.string(),
  clientAddress: z.string(),
  clientPhone: z.string(),
  currency: z.object({
    code: z.string(),
    name: z.string(),
    symbol: z.string()
  }),
  items: z.array(BillingItemSchema).min(1, 'At least one item is required'),
  subtotal: z.number(),
  tax: z.number().default(0),
  total: z.number(),
  status: z.string(),
  dueDate: z.string(),
  observation: z.string().optional()
});

// Billing Schema (output)
export const BillingSchema = CreateBillingSchema.extend({
  id: z.string(),
  pdfUrl: z.string().nullable(),
  createdAt: z.string(),
  updatedAt: z.string()
});

// Update Schema (partial)
export const UpdateBillingSchema = CreateBillingSchema.partial();

// List response with pagination
export const BillingListSchema = z.object({
  billings: z.array(BillingSchema),
  total: z.number(),
  page: z.number(),
  limit: z.number()
});
```

**Schema Organization**:
- Define base schemas first
- Use `.extend()` to add fields to existing schemas
- Use `.partial()` for update schemas
- Create separate list schemas with pagination
- Add `.openapi()` for OpenAPI metadata like examples
- Export all schemas for reuse

## Adding New Routes - Step by Step

### Step 1: Define Schema (`schemas/index.ts`)

```typescript
// Product schemas
export const ProductSchema = z.object({
  id: z.string(),
  name: z.string(),
  price: z.number(),
  createdAt: z.string()
});

export const CreateProductSchema = z.object({
  name: z.string().min(1, 'Name is required'),
  price: z.number().positive('Price must be positive')
});

export const UpdateProductSchema = CreateProductSchema.partial();

export const ProductListSchema = z.object({
  products: z.array(ProductSchema),
  total: z.number(),
  page: z.number(),
  limit: z.number()
});
```

### Step 2: Create Route Definition (`routes/product-routes.ts`)

```typescript
import { createRoute } from '@hono/zod-openapi';
import { z } from 'zod';
import { ProductSchema, CreateProductSchema, ProductListSchema, ErrorSchema } from '../schemas';

export const listProductsRoute = createRoute({
  method: 'get',
  path: '/api/products',
  tags: ['Products'],
  summary: 'List products',
  description: 'Retrieve paginated list of products',
  request: {
    query: z.object({
      page: z.string().optional().default('1'),
      limit: z.string().optional().default('10')
    })
  },
  responses: {
    200: {
      content: {
        'application/json': {
          schema: ProductListSchema
        }
      },
      description: 'Products retrieved successfully'
    }
  }
});

export const createProductRoute = createRoute({
  method: 'post',
  path: '/api/products',
  tags: ['Products'],
  summary: 'Create product',
  description: 'Create a new product',
  request: {
    body: {
      content: {
        'application/json': {
          schema: CreateProductSchema
        }
      }
    }
  },
  responses: {
    201: {
      content: {
        'application/json': {
          schema: ProductSchema
        }
      },
      description: 'Product created successfully'
    },
    400: {
      content: {
        'application/json': {
          schema: ErrorSchema
        }
      },
      description: 'Invalid input'
    }
  }
});
```

### Step 3: Implement Handler (`handlers/product-handlers.ts`)

```typescript
import type { Context } from 'hono';
import { neon } from "@neondatabase/serverless";
import { drizzle } from "drizzle-orm/neon-http";
import { products } from "../db/migrations/schema";

export const listProductsHandler = async (c: Context) => {
  try {
    const page = parseInt(c.req.query('page') || '1');
    const limit = parseInt(c.req.query('limit') || '10');

    const sql = neon(c.env.NEON_DB);
    const db = drizzle(sql);

    const productList = await db.select().from(products);

    return c.json({
      products: productList,
      total: productList.length,
      page,
      limit
    });
  } catch (error: any) {
    console.error('Error listing products:', error);
    return c.json({ error: 'Error retrieving products' }, 500);
  }
};

export const createProductHandler = async (c: Context) => {
  try {
    const body = await c.req.json();

    const sql = neon(c.env.NEON_DB);
    const db = drizzle(sql);

    const [product] = await db
      .insert(products)
      .values(body)
      .returning();

    return c.json(product, 201);
  } catch (error: any) {
    console.error('Error creating product:', error);
    return c.json({ error: 'Error creating product' }, 500);
  }
};
```

### Step 4: Register in Main App (`index.ts`)

```typescript
// Add imports at the top
import {
  listProductsRoute,
  createProductRoute
} from "./routes/product-routes";

import {
  listProductsHandler,
  createProductHandler
} from "./handlers/product-handlers";

// Register routes after other registrations
app.openapi(listProductsRoute, listProductsHandler);
app.openapi(createProductRoute, createProductHandler);
```

## Development Workflow

### 1. Start Development Server
```bash
npm run dev
```

### 2. Access Documentation
- Swagger UI: `http://localhost:8787/docs`
- OpenAPI JSON: `http://localhost:8787/docs/openapi.json`
- Health Check: `http://localhost:8787/health`

### 3. Test Endpoints
Use the Swagger UI to test endpoints interactively, or use curl/Postman.

### 4. Deploy
```bash
npm run deploy
```

## Best Practices

### 1. Separation of Concerns
- **Routes**: OpenAPI specification only
- **Handlers**: Business logic and data operations
- **Schemas**: Validation and type definitions
- **Services**: Reusable business logic

### 2. Type Safety
- Define environment bindings in `Env` type
- Use Zod schemas for runtime validation
- Leverage TypeScript for compile-time checks

### 3. Error Handling
```typescript
// Consistent error response format
return c.json({
  error: 'User-friendly message',
  code: 'ERROR_CODE'
}, statusCode);
```

### 4. Naming Conventions
- Routes: `*Route` (e.g., `createBillingRoute`)
- Handlers: `*Handler` (e.g., `createBillingHandler`)
- Schemas: `*Schema` (e.g., `CreateBillingSchema`)

### 5. Documentation
- Use descriptive `summary` and `description` in routes
- Add examples to schemas using `.openapi({ example: '...' })`
- Include all possible response codes
- Tag related endpoints for Swagger grouping

### 6. Middleware Organization
```typescript
// Order matters!
app.use('*', logger());        // 1. Logging
app.use('*', prettyJSON());    // 2. Response formatting
app.use('*', cors());          // 3. CORS
// 4. Setup Swagger
// 5. Register routes
```

### 7. Environment Variables
- Store credentials in Cloudflare secrets
- Use environment variables for configuration
- Provide sensible defaults for development

## Advanced Patterns

### Authentication Middleware

```typescript
import { jwt } from 'hono/jwt';

// Protect specific routes
app.use('/api/*', jwt({
  secret: c.env.JWT_SECRET
}));

// Or apply to individual routes in the route definition
export const protectedRoute = createRoute({
  method: 'get',
  path: '/api/protected',
  security: [{ bearerAuth: [] }],
  // ... rest of route
});
```

### Request Validation

```typescript
// Validation happens automatically through Zod schemas
// Access validated data in handler:
export const handler = async (c: Context) => {
  const validated = c.req.valid('json'); // Already validated!
  // Use validated data safely
};
```

### Response Formatting

```typescript
// Use consistent response wrappers
export const successResponse = (data: any, message?: string) => ({
  success: true,
  message,
  data
});

export const errorResponse = (error: string, code?: string) => ({
  success: false,
  error,
  code
});
```

## Benefits of This Architecture

✅ **Auto-generated OpenAPI documentation** from route definitions
✅ **Type-safe** request/response handling throughout
✅ **Built-in validation** with Zod schemas
✅ **Interactive Swagger UI** out of the box
✅ **Clear separation** between routing and business logic
✅ **Easy to test** handlers independently
✅ **Scalable** structure for growing APIs
✅ **Self-documenting** code
✅ **Edge-ready** for Cloudflare Workers

## Common Patterns

### Pagination
```typescript
const query = z.object({
  page: z.string().optional().default('1'),
  limit: z.string().optional().default('10')
});
```

### Filtering
```typescript
const query = z.object({
  status: z.enum(['active', 'inactive']).optional(),
  search: z.string().optional()
});
```

### Nested Resources
```typescript
// GET /api/clients/:clientId/billings
path: '/api/clients/{clientId}/billings'
params: z.object({
  clientId: z.string().uuid()
})
```

### File Uploads (Binary)
```typescript
responses: {
  200: {
    content: {
      'application/pdf': {
        schema: { type: 'string', format: 'binary' }
      }
    }
  }
}
```

## Troubleshooting

### Schema Validation Errors
- Check that Zod schema matches actual data structure
- Ensure required fields are provided
- Verify data types (string vs number)

### Routes Not Showing in Swagger
- Ensure route is registered with `app.openapi()`
- Check that `setupSwagger()` is called before route registration
- Verify tags are correctly specified

### Type Errors
- Run `npm run cf-typegen` to update Cloudflare bindings
- Ensure `Env` type includes all necessary bindings
- Check that handler signatures match route definitions

## Resources

- [Hono Documentation](https://hono.dev)
- [@hono/zod-openapi](https://github.com/honojs/middleware/tree/main/packages/zod-openapi)
- [Zod Documentation](https://zod.dev)
- [Cloudflare Workers Docs](https://developers.cloudflare.com/workers/)
- [OpenAPI Specification](https://swagger.io/specification/)

---

**This architecture pattern enables rapid API development with excellent type safety, automatic documentation, and production-ready deployment on Cloudflare's edge network.**