# NODE.md - Node.js Development Guide

## 📦 Package Manager

**CRITICAL: Always use the same package manager consistently**
- Check for `yarn.lock` → use `yarn`
- Check for `pnpm-lock.yaml` → use `pnpm`
- Check for `package-lock.json` → use `npm`
- **NEVER mix** package managers in the same project

## 🏗️ Project Structure

```
src/
├── controllers/      # Request handlers (HTTP layer)
├── services/         # Business logic
├── repositories/     # Database operations
├── routes/           # API routes
├── middlewares/      # Express middlewares
├── utils/            # Utility functions
├── config/           # Configuration
└── types/            # TypeScript types

tests/
├── unit/             # Unit tests
├── integration/      # API tests
└── fixtures/         # Test data
```

### File Size Rules
- Keep files under **300 lines**
- One file = one responsibility
- Extract when growing beyond 300 lines

## 🔒 Environment Variables

```env
# Server
NODE_ENV=development
PORT=3000

# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/db
DB_POOL_MAX=10

# Authentication
JWT_SECRET=your-secret-key-min-32-chars
JWT_EXPIRES_IN=7d

# External Services
REDIS_URL=redis://localhost:6379
SMTP_HOST=smtp.gmail.com

# API Keys (server-only)
API_SECRET_KEY=
AWS_ACCESS_KEY_ID=
```

### Security Rules
- **NEVER commit `.env` files** - add to `.gitignore`
- Provide `.env.example` with empty values
- Validate required variables on startup:

```typescript
// config/env.ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']),
  PORT: z.string().transform(Number),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32)
});

export const env = envSchema.parse(process.env);
```

## 🚀 Error Handling (CRITICAL)

### Custom Error Class

```typescript
// utils/errors.ts
export class AppError extends Error {
  constructor(
    public statusCode: number,
    public message: string,
    public isOperational = true
  ) {
    super(message);
    Error.captureStackTrace(this, this.constructor);
  }
}
```

### Global Error Handler

```typescript
// middlewares/errorHandler.ts
export const errorHandler = (err: Error, req: Request, res: Response, next: NextFunction) => {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      status: 'error',
      message: err.message
    });
  }

  // Don't expose internal errors in production
  console.error('Unexpected Error:', err);
  return res.status(500).json({
    status: 'error',
    message: process.env.NODE_ENV === 'production' ? 'Internal server error' : err.message
  });
};
```

### Async Handler

```typescript
// utils/asyncHandler.ts
export const asyncHandler = (fn: Function) => {
  return (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
};

// Usage
app.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await userService.findById(req.params.id);
  if (!user) throw new AppError(404, 'User not found');
  res.json({ data: user });
}));
```

### Graceful Shutdown

```typescript
// server.ts
process.on('SIGTERM', async () => {
  server.close(async () => {
    await db.close();
    await redis.quit();
    process.exit(0);
  });
});
```

### Rules:
- ✅ Always handle errors explicitly
- ✅ Use try-catch for async operations
- ✅ Log errors with context
- ❌ Never silent failures
- ❌ Never expose stack traces in production

## 🔐 Security Best Practices

### Input Validation

```typescript
import { z } from 'zod';

export const createUserSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  name: z.string().min(2).max(50)
});

// Middleware
export const validate = (schema: z.ZodSchema) => {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      schema.parse(req.body);
      next();
    } catch (error) {
      throw new AppError(400, 'Validation failed');
    }
  };
};
```

### Security Middleware

```typescript
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';
import cors from 'cors';

// Security headers
app.use(helmet());

// CORS
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(','),
  credentials: true
}));

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // 100 requests per IP
});
app.use('/api/', limiter);
```

### Password & Authentication

```typescript
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';

// Hash password
export const hashPassword = async (password: string) => {
  return bcrypt.hash(password, 12);
};

// Compare password
export const comparePassword = async (password: string, hash: string) => {
  return bcrypt.compare(password, hash);
};

// Generate JWT
export const generateToken = (payload: object) => {
  return jwt.sign(payload, process.env.JWT_SECRET!, {
    expiresIn: '7d'
  });
};

// Verify JWT
export const verifyToken = (token: string) => {
  return jwt.verify(token, process.env.JWT_SECRET!);
};
```

### SQL Injection Prevention

```typescript
// ✅ ALWAYS use parameterized queries
db.query('SELECT * FROM users WHERE email = $1', [email]);

// ❌ NEVER concatenate user input
db.query(`SELECT * FROM users WHERE email = '${email}'`); // DANGEROUS!
```

## 📊 Logging

```typescript
// utils/logger.ts
import winston from 'winston';

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/combined.log' })
  ]
});

// Console in development
if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.simple()
  }));
}

// Usage
logger.info('User created', { userId, email });
logger.error('Database error', { error: err.message });
```

### What to Log
- ✅ Application startup/shutdown
- ✅ Authentication attempts
- ✅ Critical errors with stack traces
- ✅ External API calls
- ✅ Performance metrics

### Never Log
- ❌ Passwords or tokens
- ❌ Credit card numbers
- ❌ Personal data (PII)
- ❌ API keys

## 🗄️ Database Best Practices

### Connection Pooling

```typescript
// config/database.ts
import { Pool } from 'pg';

export const pool = new Pool({
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT || '5432'),
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 10, // Maximum pool size
  idleTimeoutMillis: 30000
});

// Graceful shutdown
export const closeDB = async () => {
  await pool.end();
};
```

### SQL Safety

```typescript
// ✅ ALWAYS parameterized queries
const user = await pool.query(
  'SELECT * FROM users WHERE email = $1',
  [email]
);

// ❌ NEVER concatenate
const user = await pool.query(
  `SELECT * FROM users WHERE email = '${email}'` // SQL INJECTION!
);
```

### Transactions

```typescript
export const transferFunds = async (fromId: string, toId: string, amount: number) => {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
      [amount, fromId]
    );
    
    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
      [amount, toId]
    );
    
    await client.query('COMMIT');
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
};
```

## ⚡ Performance Optimization

### Caching with Redis

```typescript
// utils/cache.ts
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

export const cache = {
  async get<T>(key: string): Promise<T | null> {
    const data = await redis.get(key);
    return data ? JSON.parse(data) : null;
  },
  
  async set(key: string, value: any, ttl: number = 3600): Promise<void> {
    await redis.setex(key, ttl, JSON.stringify(value));
  },
  
  async del(key: string): Promise<void> {
    await redis.del(key);
  }
};

// Usage
const getUser = async (userId: string) => {
  const cacheKey = `user:${userId}`;
  
  let user = await cache.get(cacheKey);
  if (user) return user;
  
  user = await userRepository.findById(userId);
  await cache.set(cacheKey, user, 3600);
  
  return user;
};
```

### Avoid Blocking Operations

```typescript
// ❌ BAD: Blocking sync operations
import fs from 'fs';
const data = fs.readFileSync('./file.txt'); // BLOCKS!

// ✅ GOOD: Non-blocking async
import fs from 'fs/promises';
const data = await fs.readFile('./file.txt');

// ✅ GOOD: Use streams for large files
import { createReadStream } from 'fs';
const stream = createReadStream('./large-file.txt');
```

### Worker Threads for CPU Tasks

```typescript
// For CPU-intensive tasks
import { Worker } from 'worker_threads';

const runHeavyTask = (data: any): Promise<any> => {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./workers/process.js');
    worker.postMessage(data);
    worker.on('message', resolve);
    worker.on('error', reject);
  });
};
```

## 🧪 Testing

### Test Structure
```
tests/
├── unit/           # Service & util tests
├── integration/    # API tests
└── fixtures/       # Test data
```

### Unit Test Example

```typescript
// tests/unit/userService.test.ts
import { describe, it, expect, vi } from 'vitest';
import { userService } from '@/services/userService';
import { userRepository } from '@/repositories/userRepository';

vi.mock('@/repositories/userRepository');

describe('UserService', () => {
  it('should create user with hashed password', async () => {
    const userData = {
      email: 'test@example.com',
      password: 'Password123',
      name: 'Test User'
    };

    vi.mocked(userRepository.findByEmail).mockResolvedValue(null);
    vi.mocked(userRepository.create).mockResolvedValue({
      id: '123',
      ...userData,
      password: 'hashed'
    });

    const user = await userService.create(userData);

    expect(user.password).not.toBe(userData.password);
    expect(userRepository.findByEmail).toHaveBeenCalledWith(userData.email);
  });
});
```

### Integration Test Example

```typescript
// tests/integration/users.test.ts
import { describe, it, expect } from 'vitest';
import request from 'supertest';
import app from '@/app';

describe('User API', () => {
  it('should return 401 when not authenticated', async () => {
    await request(app)
      .get('/api/users/123')
      .expect(401);
  });

  it('should return user when authenticated', async () => {
    const response = await request(app)
      .get('/api/users/123')
      .set('Authorization', `Bearer ${token}`)
      .expect(200);

    expect(response.body.data).toHaveProperty('id');
  });
});
```

## 🔄 API Design

### RESTful Conventions

```typescript
// Use plural nouns for resources
router.get('/users', getUsers);           // List
router.get('/users/:id', getUser);        // Get one
router.post('/users', createUser);        // Create
router.put('/users/:id', updateUser);     // Update (full)
router.patch('/users/:id', patchUser);    // Update (partial)
router.delete('/users/:id', deleteUser);  // Delete

// Nested resources
router.get('/users/:userId/posts', getUserPosts);

// Actions
router.post('/users/:id/activate', activateUser);
```

### Consistent Response Format

```typescript
// Success
res.json({
  status: 'success',
  data: user
});

// Error
res.status(400).json({
  status: 'error',
  message: 'Invalid input'
});

// Paginated
res.json({
  status: 'success',
  data: users,
  pagination: {
    page: 1,
    limit: 10,
    total: 100,
    totalPages: 10
  }
});
```

### API Versioning

```typescript
// Version in URL (recommended)
app.use('/api/v1', v1Routes);
app.use('/api/v2', v2Routes);
```

## 🎯 Core Principles Summary

1. **Package Manager**: Never mix npm/yarn/pnpm
2. **Error Handling**: Always handle errors, use custom error classes
3. **Security**: Validate input, parameterized queries, hash passwords
4. **Environment Variables**: Never commit secrets, validate on startup
5. **Logging**: Log important events, never log sensitive data
6. **Database**: Use connection pooling, parameterized queries, transactions
7. **Performance**: Cache data, avoid blocking, use worker threads
8. **Testing**: Test business logic, mock dependencies
9. **Code Quality**: Keep files <300 lines, single responsibility
10. **API Design**: RESTful conventions, versioning, consistent responses

---

**Remember**: These practices ensure your Node.js application is **secure**, **maintainable**, **performant**, and **production-ready**.