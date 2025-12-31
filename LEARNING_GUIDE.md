# 🎓 Ultimate NestJS Boilerplate - Learning Guide

Welcome! This guide will help you understand the architecture, patterns, and best practices used in this NestJS boilerplate. It's designed for developers with intermediate/basic NestJS knowledge who want to learn proper structure and paradigms.

## 📚 Table of Contents

1. [Quick Start](#quick-start)
2. [Project Structure Overview](#project-structure-overview)
3. [Core Architecture Concepts](#core-architecture-concepts)
4. [Learning Path by Module](#learning-path-by-module)
5. [Design Patterns & Best Practices](#design-patterns--best-practices)
6. [Common Workflows](#common-workflows)
7. [Testing Strategy](#testing-strategy)
8. [Advanced Features](#advanced-features)
9. [Development Tips](#development-tips)

---

## 🚀 Quick Start

### Prerequisites
- Node.js (see `.nvmrc` for version)
- pnpm package manager
- Docker & Docker Compose
- Basic understanding of TypeScript and NestJS

### Initial Setup
```bash
# 1. Setup environment files
cp .env.example .env
cp .env.docker.example .env.docker

# 2. Start Docker containers (PostgreSQL, Redis, etc.)
pnpm docker:dev:up

# 3. Run database migrations
docker exec -it nestjs-boilerplate-server sh
pnpm migration:up
exit

# 4. Start development server
pnpm start:dev
```

### What Just Happened?
- **Docker containers**: Spun up PostgreSQL, Redis, MailPit, and optionally Prometheus/Grafana
- **Migrations**: Created database tables based on TypeORM entities
- **Dev server**: Started with hot-reload and email template watching

---

## 📂 Project Structure Overview

```
src/
├── api/                    # REST API endpoints (feature modules)
│   ├── user/              # User management feature
│   ├── file/              # File upload feature
│   └── health/            # Health check endpoints
│
├── auth/                   # Authentication module (Better Auth)
│   ├── entities/          # Auth-related entities
│   ├── auth.service.ts    # Auth business logic
│   └── better-auth.service.ts
│
├── common/                 # Shared DTOs and types
│   ├── dto/               # Reusable DTOs (pagination, etc.)
│   └── types/             # Common TypeScript types
│
├── config/                 # Configuration modules
│   ├── app/               # App-wide configuration
│   ├── database/          # Database configuration
│   ├── redis/             # Redis configuration
│   └── ...                # Other service configs
│
├── database/              # Database-related files
│   ├── models/            # Base models for entities
│   ├── migrations/        # TypeORM migrations
│   └── seeds/             # Database seeders
│
├── decorators/            # Custom decorators
│   ├── auth/              # Auth-specific decorators
│   ├── validators/        # Custom validation decorators
│   └── *.decorators.ts    # Various decorators
│
├── graphql/               # GraphQL configuration
│
├── i18n/                  # Internationalization
│   └── translations/      # Translation files (en, es, etc.)
│
├── interceptors/          # Global interceptors
│
├── middlewares/           # Custom middlewares
│
├── services/              # Shared services
│   ├── aws/               # AWS S3 service
│   └── gcp/               # GCP services
│
├── shared/                # Shared modules
│   ├── cache/             # Redis cache module
│   ├── mail/              # Email service with React Email
│   └── socket/            # WebSocket module
│
├── tools/                 # Development tools
│   ├── swagger/           # Swagger/OpenAPI setup
│   ├── logger/            # Pino logger setup
│   └── grafana/           # Monitoring dashboards
│
├── utils/                 # Utility functions
│   ├── pagination/        # Pagination helpers
│   └── validators/        # Custom validators
│
├── worker/                # Background job processing
│   └── queues/            # BullMQ queue processors
│
├── app.module.ts          # Root application module
└── main.ts                # Application entry point
```

### Key Structural Principles

1. **Feature-based organization**: API endpoints are organized by feature (user, file, etc.)
2. **Separation of concerns**: Clear separation between API, business logic, data access, and infrastructure
3. **Shared resources**: Common utilities, DTOs, and services are centralized
4. **Configuration management**: All configs are type-safe and validated
5. **Modular architecture**: Each feature is a self-contained module

---

## 🏗️ Core Architecture Concepts

### 1. Module System

NestJS uses a modular architecture. Here's how this boilerplate structures modules:

```typescript
// Example: User Module (src/api/user/user.module.ts)
@Module({
  imports: [TypeOrmModule.forFeature([UserEntity])],  // Dependencies
  controllers: [UserController],                       // HTTP handlers
  providers: [UserService, UserResolver],             // Business logic
  exports: [UserService],                              // Exposed services
})
export class UserModule {}
```

**Key Concepts:**
- **Imports**: Bring in other modules/features
- **Controllers**: Handle HTTP/REST requests
- **Providers**: Injectable services (business logic)
- **Exports**: Make services available to other modules

### 2. Dependency Injection (DI)

This boilerplate extensively uses DI for loose coupling:

```typescript
// Services are injected via constructor
@Injectable()
export class UserService {
  constructor(
    private readonly i18nService: I18nService,           // Translation service
    @InjectRepository(UserEntity)                         // Database repository
    private readonly userRepository: Repository<UserEntity>,
    private readonly betterAuthService: BetterAuthService, // Auth service
  ) {}
}
```

**Benefits:**
- Testability (easy to mock dependencies)
- Loose coupling
- Single Responsibility Principle

### 3. Configuration Pattern

Configuration is centralized, validated, and type-safe:

```typescript
// src/config/app/app.config.ts
class EnvironmentVariablesValidator {
  @IsEnum(Environment)
  NODE_ENV: typeof Environment;
  
  @IsInt()
  @Min(0)
  @Max(65535)
  APP_PORT: number;
}

export default registerAs<AppConfig>('app', () => {
  validateConfig(process.env, EnvironmentVariablesValidator);
  return getConfig();
});
```

**Pattern Benefits:**
- Runtime validation
- Type safety
- Single source of truth
- Easy testing

### 4. DTO (Data Transfer Object) Pattern

DTOs define the shape of data for API requests/responses:

```typescript
// src/api/user/dto/user.dto.ts
export class UserDto {
  @ApiProperty()
  id: string;

  @ApiProperty()
  email: string;

  @ApiProperty({ required: false })
  @IsOptional()
  username?: string;
}

export class UpdateUserProfileDto {
  @ApiPropertyOptional()
  @IsString()
  @IsOptional()
  firstName?: string;

  @ApiPropertyOptional()
  @IsString()
  @IsOptional()
  lastName?: string;
}
```

**DTO Benefits:**
- Input validation
- API documentation (via decorators)
- Type safety
- Data transformation

### 5. Repository Pattern

Database access is abstracted through TypeORM repositories:

```typescript
// In UserService
constructor(
  @InjectRepository(UserEntity)
  private readonly userRepository: Repository<UserEntity>,
) {}

async findOneUser(id: Uuid): Promise<UserDto> {
  const user = await this.userRepository.findOne({ where: { id } });
  if (!user) {
    throw new NotFoundException(this.i18nService.t('user.notFound'));
  }
  return user;
}
```

**Repository Pattern Benefits:**
- Separation of data access logic
- Easier to test
- Database abstraction

---

## 🎯 Learning Path by Module

### Level 1: Foundation (Start Here)

#### 1.1 Health Check Module (`src/api/health/`)
**Why start here?** Simplest module with minimal dependencies.

**What to learn:**
- Basic controller structure
- Simple DTOs
- Terminus health checks
- Testing with Jest

**Files to study:**
```
src/api/health/
├── health.controller.ts       # HTTP endpoints
├── health.module.ts           # Module definition
├── dto/health.dto.ts          # Response DTOs
└── health.controller.spec.ts  # Unit tests
```

**Key patterns:**
```typescript
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('database'),
    ]);
  }
}
```

#### 1.2 Configuration System (`src/config/`)
**What to learn:**
- Environment variable validation
- Type-safe configuration
- registerAs pattern
- Factory providers

**Study this flow:**
1. `.env` file → Environment variables
2. `app.config.ts` → Validates with `class-validator`
3. `ConfigService` → Type-safe access throughout app

#### 1.3 Main Application Setup (`src/main.ts`)
**What to learn:**
- Application bootstrap
- Middleware setup
- Global pipes and interceptors
- Fastify adapter usage
- Graceful shutdown

**Key concepts in main.ts:**
```typescript
// 1. Create NestJS app with Fastify
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule.main(),
  new FastifyAdapter(),
);

// 2. Global validation pipe
app.useGlobalPipes(
  new ValidationPipe({
    transform: true,
    whitelist: true,
    forbidNonWhitelisted: true,
  }),
);

// 3. API versioning
app.enableVersioning({ type: VersioningType.URI });

// 4. CORS configuration
app.enableCors({ /* ... */ });

// 5. Security headers
app.use(helmet());
```

### Level 2: Core Features

#### 2.1 User Module (`src/api/user/`)
**What to learn:**
- Complete CRUD operations
- REST and GraphQL resolvers
- Pagination (offset & cursor-based)
- Service layer patterns
- Custom decorators

**Files to study:**
```
src/api/user/
├── user.controller.ts    # REST endpoints
├── user.service.ts       # Business logic
├── user.resolver.ts      # GraphQL resolver
├── user.module.ts        # Module config
├── dto/
│   ├── user.dto.ts       # Response DTOs
│   └── update-user-profile.dto.ts
└── schema/               # GraphQL schemas
```

**Key patterns:**

1. **Controller with Guards:**
```typescript
@ApiTags('user')
@Controller({ path: 'user', version: '1' })
@UseGuards(AuthGuard)  // Global auth protection
export class UserController {
  @Get('whoami')
  @ApiAuth({ summary: 'Get current user', type: UserDto })
  async getCurrentUser(
    @CurrentUserSession('user') user: CurrentUserSession['user'],
  ): Promise<UserDto> {
    return await this.userService.findOneUser(user.id);
  }
}
```

2. **Pagination (Offset-based):**
```typescript
async findAllUsers(dto: QueryUsersOffsetDto): Promise<OffsetPaginatedDto<UserDto>> {
  const query = this.userRepository
    .createQueryBuilder('user')
    .orderBy('user.createdAt', 'DESC');
  
  const [users, metaDto] = await paginate<UserEntity>(query, dto);
  return new OffsetPaginatedDto(users, metaDto);
}
```

3. **Pagination (Cursor-based):**
```typescript
async findAllUsersCursor(reqDto: QueryUsersCursorDto) {
  const queryBuilder = this.userRepository.createQueryBuilder('user');
  const paginator = buildPaginator({
    entity: UserEntity,
    alias: 'user',
    paginationKeys: ['createdAt'],
    query: {
      limit: reqDto.limit,
      order: 'DESC',
      afterCursor: reqDto.afterCursor,
      beforeCursor: reqDto.beforeCursor,
    },
  });
  const { data, cursor } = await paginator.paginate(queryBuilder);
  return new CursorPaginatedDto(data, new CursorPaginationDto(/*...*/));
}
```

#### 2.2 Authentication (`src/auth/`)
**What to learn:**
- Better Auth integration
- Custom auth guards
- Session management
- Auth decorators
- Hooks system

**Study:**
```
src/auth/
├── auth.module.ts           # Better Auth setup
├── auth.service.ts          # Auth business logic
├── better-auth.service.ts   # Better Auth wrapper
├── auth.guard.ts            # Auth guard implementation
└── entities/user.entity.ts  # User entity
```

**Key patterns:**

1. **Custom Decorator for Current User:**
```typescript
// src/decorators/auth/current-user-session.decorator.ts
export const CurrentUserSession = createParamDecorator(
  (data: keyof CurrentUserSession, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return data ? request.userSession?.[data] : request.userSession;
  },
);
```

2. **Auth Guard:**
```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const session = await this.betterAuthService.api.getSession({
      headers: request.headers,
    });
    
    if (!session?.user) {
      throw new UnauthorizedException();
    }
    
    request.userSession = session;
    return true;
  }
}
```

#### 2.3 File Upload (`src/api/file/`)
**What to learn:**
- Multipart file handling
- Local vs S3 storage
- File validation
- Streaming uploads

**Key concepts:**
```typescript
@Post('upload')
@ApiAuth({ summary: 'Upload file' })
async uploadFile(
  @UploadedFile() file: Express.Multer.File,
): Promise<FileDto> {
  return this.fileService.upload(file);
}
```

### Level 3: Advanced Concepts

#### 3.1 Background Jobs (`src/worker/`)
**What to learn:**
- BullMQ queue setup
- Job processors
- Queue monitoring (Bull Board)
- Worker vs API server separation

**Study:**
```
src/worker/
├── worker.module.ts           # Worker module
└── queues/
    └── email/
        ├── email.module.ts
        ├── email.processor.ts  # Job handler
        └── email.service.ts    # Queue producer
```

**Pattern:**
```typescript
// Producer (in API server)
@Injectable()
export class EmailService {
  constructor(@InjectQueue(Queue.Email) private emailQueue: Queue) {}

  async sendEmail(data: SendEmailDto) {
    await this.emailQueue.add('send-email', data);
  }
}

// Consumer (in Worker server)
@Processor(Queue.Email)
export class EmailProcessor {
  @Process('send-email')
  async handleSendEmail(job: Job<SendEmailDto>) {
    // Process email sending
  }
}
```

#### 3.2 Caching (`src/shared/cache/`)
**What to learn:**
- Redis cache integration
- Cache decorators
- Cache invalidation patterns
- TTL management

**Usage:**
```typescript
@Injectable()
export class CacheService {
  constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}

  async get<T>(key: string): Promise<T | undefined> {
    return await this.cacheManager.get<T>(key);
  }

  async set(key: string, value: any, ttl?: number): Promise<void> {
    await this.cacheManager.set(key, value, ttl);
  }
}
```

#### 3.3 WebSockets (`src/shared/socket/`)
**What to learn:**
- Socket.io integration
- Redis adapter for scaling
- Room management
- Real-time events

**Pattern:**
```typescript
@WebSocketGateway({
  cors: { origin: '*' },
  transports: ['websocket'],
})
export class SocketGateway {
  @WebSocketServer()
  server: Server;

  @SubscribeMessage('message')
  handleMessage(client: Socket, payload: any): void {
    this.server.emit('message', payload);
  }
}
```

#### 3.4 Email Templates (`src/shared/mail/`)
**What to learn:**
- React Email for templates
- Template compilation
- Multi-language support
- Email testing with MailPit

**Development workflow:**
```bash
# 1. Edit templates in src/shared/mail/templates/*.tsx
# 2. Watch mode compiles to .hbs
pnpm email:watch

# 3. Preview in browser
pnpm email:dev

# 4. Test in MailPit at http://localhost:18025
```

#### 3.5 GraphQL (`src/graphql/` & resolvers)
**What to learn:**
- Code-first approach
- Schema generation
- Query/Mutation resolvers
- GraphQL + REST coexistence

**Pattern:**
```typescript
@Resolver(() => UserSchema)
export class UserResolver {
  constructor(private readonly userService: UserService) {}

  @Query(() => UserSchema)
  async user(@Args('id') id: string): Promise<UserDto> {
    return this.userService.findOneUser(id);
  }

  @Mutation(() => UserSchema)
  async deleteUser(@Args('id') id: string) {
    return this.userService.deleteUser(id);
  }
}
```

---

## 🎨 Design Patterns & Best Practices

### 1. Custom Decorators

The boilerplate uses decorators extensively for cleaner code:

#### Auth Decorators
```typescript
// Get current user session
@Get('profile')
async getProfile(@CurrentUserSession('user') user: User) {
  // user is automatically injected
}

// Optional authentication
@OptionalAuth()
@Get('public-or-private')
async flexibleEndpoint(@CurrentUserSession() session?: CurrentUserSession) {
  // Works with or without auth
}
```

#### API Documentation Decorators
```typescript
@ApiAuth({
  summary: 'Get current user',
  type: UserDto,
  errorResponses: [401, 404],
})
@Get('whoami')
async getCurrentUser() { /* ... */ }
```

#### Validation Decorators
```typescript
export class CreateUserDto {
  @IsPassword()  // Custom password validator
  @IsString()
  password: string;

  @IsNullable()  // Allows null values
  @IsString()
  middleName?: string | null;
}
```

### 2. Error Handling

Consistent error handling across the application:

```typescript
// Use built-in exceptions
if (!user) {
  throw new NotFoundException(this.i18nService.t('user.notFound'));
}

// HTTP status codes are automatic
throw new UnauthorizedException('Invalid credentials');
throw new ForbiddenException('Access denied');
throw new BadRequestException('Invalid input');
```

### 3. Internationalization (i18n)

Multi-language support built-in:

```typescript
// In services
throw new NotFoundException(
  this.i18nService.t('user.notFound')  // Uses locale from request
);

// Translation files: src/i18n/translations/en/user.json
{
  "notFound": "User not found",
  "created": "User created successfully"
}
```

### 4. Validation Pipes

Automatic validation on all endpoints:

```typescript
export class CreateUserDto {
  @IsEmail()
  @IsNotEmpty()
  email: string;

  @IsString()
  @MinLength(8)
  @MaxLength(100)
  password: string;

  @IsOptional()
  @IsString()
  firstName?: string;
}

// Automatically validates on POST /user
@Post()
async create(@Body() dto: CreateUserDto) {
  // dto is guaranteed to be valid
}
```

### 5. Database Migrations

TypeORM migrations for version control of schema:

```bash
# Generate migration from entity changes
pnpm migration:generate src/database/migrations/AddUserFields

# Run pending migrations
pnpm migration:up

# Revert last migration
pnpm migration:down

# View migration status
pnpm migration:show
```

### 6. Soft Deletes

Entities use soft deletes to preserve data:

```typescript
// In UserService
async deleteUser(id: string) {
  await this.userRepository.softDelete(id);  // Sets deletedAt timestamp
  return HttpStatus.OK;
}

// Query excludes soft-deleted by default
// To include them:
await this.userRepository.find({ withDeleted: true });
```

### 7. API Versioning

URI-based versioning for backward compatibility:

```typescript
@Controller({ path: 'user', version: '1' })  // /api/v1/user
export class UserController {}

@Controller({ path: 'user', version: '2' })  // /api/v2/user
export class UserV2Controller {}
```

### 8. Rate Limiting

Redis-backed rate limiting:

```typescript
// Global configuration in app.module.ts
ThrottlerModule.forRootAsync({
  useFactory: () => ({
    throttlers: [
      {
        ttl: 60000,    // 1 minute
        limit: 100,    // 100 requests
      },
    ],
  }),
})

// Custom limits per endpoint
@Throttle({ default: { limit: 5, ttl: 60000 } })
@Post('login')
async login() {}
```

---

## 🔄 Common Workflows

### 1. Adding a New Feature Module

```bash
# Example: Adding a "Post" feature

# 1. Create module structure
mkdir -p src/api/post/{dto,entities}

# 2. Create entity
# src/database/entities/post.entity.ts
@Entity('posts')
export class PostEntity extends BaseModel {
  @Column()
  title: string;

  @Column('text')
  content: string;

  @ManyToOne(() => UserEntity)
  author: UserEntity;
}

# 3. Create DTOs
# src/api/post/dto/post.dto.ts
export class PostDto {
  @ApiProperty()
  id: string;

  @ApiProperty()
  title: string;

  @ApiProperty()
  content: string;
}

export class CreatePostDto {
  @IsString()
  @IsNotEmpty()
  title: string;

  @IsString()
  @IsNotEmpty()
  content: string;
}

# 4. Create service
# src/api/post/post.service.ts
@Injectable()
export class PostService {
  constructor(
    @InjectRepository(PostEntity)
    private postRepository: Repository<PostEntity>,
  ) {}

  async create(dto: CreatePostDto): Promise<PostDto> {
    const post = this.postRepository.create(dto);
    return await this.postRepository.save(post);
  }
}

# 5. Create controller
# src/api/post/post.controller.ts
@ApiTags('post')
@Controller({ path: 'post', version: '1' })
export class PostController {
  constructor(private readonly postService: PostService) {}

  @Post()
  @ApiAuth({ summary: 'Create post', type: PostDto })
  async create(@Body() dto: CreatePostDto) {
    return this.postService.create(dto);
  }
}

# 6. Create module
# src/api/post/post.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([PostEntity])],
  controllers: [PostController],
  providers: [PostService],
  exports: [PostService],
})
export class PostModule {}

# 7. Register in ApiModule
# src/api/api.module.ts
@Module({
  imports: [
    UserModule,
    FileModule,
    PostModule,  // Add here
  ],
})
export class ApiModule {}

# 8. Generate and run migration
pnpm migration:generate src/database/migrations/CreatePostsTable
pnpm migration:up
```

### 2. Adding a Background Job

```typescript
// 1. Create queue name constant
// src/constants/job.constant.ts
export enum Queue {
  Email = 'email',
  Notification = 'notification',  // Add new queue
}

// 2. Create processor
// src/worker/queues/notification/notification.processor.ts
@Processor(Queue.Notification)
export class NotificationProcessor {
  @Process('send-push')
  async handlePushNotification(job: Job<{ userId: string; message: string }>) {
    // Process notification
    console.log(`Sending to ${job.data.userId}: ${job.data.message}`);
  }
}

// 3. Create producer service
// src/worker/queues/notification/notification.service.ts
@Injectable()
export class NotificationService {
  constructor(
    @InjectQueue(Queue.Notification) private notificationQueue: Queue
  ) {}

  async sendPush(userId: string, message: string) {
    await this.notificationQueue.add('send-push', { userId, message });
  }
}

// 4. Register in modules
// Add to worker.module.ts and any API module that needs to enqueue jobs
```

### 3. Adding Configuration

```typescript
// 1. Create config file
// src/config/payment/payment.config.ts
import { registerAs } from '@nestjs/config';
import { IsString, IsNotEmpty } from 'class-validator';
import validateConfig from '@/utils/config/validate-config';

class EnvironmentVariablesValidator {
  @IsString()
  @IsNotEmpty()
  PAYMENT_API_KEY: string;

  @IsString()
  @IsNotEmpty()
  PAYMENT_API_SECRET: string;
}

export type PaymentConfig = {
  apiKey: string;
  apiSecret: string;
};

export default registerAs<PaymentConfig>('payment', () => {
  validateConfig(process.env, EnvironmentVariablesValidator);
  return {
    apiKey: process.env.PAYMENT_API_KEY,
    apiSecret: process.env.PAYMENT_API_SECRET,
  };
});

// 2. Add to app.module.ts
ConfigModule.forRoot({
  load: [
    appConfig,
    paymentConfig,  // Add here
  ],
});

// 3. Add to config.type.ts
export type GlobalConfig = {
  app: AppConfig;
  payment: PaymentConfig;  // Add type
  // ...
};

// 4. Use in services
constructor(private configService: ConfigService<GlobalConfig>) {}

const apiKey = this.configService.get('payment.apiKey', { infer: true });
```

---

## 🧪 Testing Strategy

### Unit Tests

```typescript
// Example: src/api/user/user.service.spec.ts
describe('UserService', () => {
  let service: UserService;
  let repository: Repository<UserEntity>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UserService,
        {
          provide: getRepositoryToken(UserEntity),
          useValue: {
            findOne: jest.fn(),
            save: jest.fn(),
          },
        },
        {
          provide: I18nService,
          useValue: { t: jest.fn() },
        },
      ],
    }).compile();

    service = module.get<UserService>(UserService);
    repository = module.get(getRepositoryToken(UserEntity));
  });

  it('should find a user by id', async () => {
    const user = { id: '123', email: 'test@example.com' };
    jest.spyOn(repository, 'findOne').mockResolvedValue(user as any);

    const result = await service.findOneUser('123');
    expect(result).toEqual(user);
  });
});
```

### E2E Tests

```typescript
// Example: test/user.e2e-spec.ts
describe('UserController (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  it('/api/v1/user (GET)', () => {
    return request(app.getHttpServer())
      .get('/api/v1/user/all')
      .set('Authorization', 'Bearer token')
      .expect(200)
      .expect((res) => {
        expect(res.body.data).toBeInstanceOf(Array);
      });
  });
});
```

### Running Tests

```bash
# Unit tests
pnpm test

# Watch mode
pnpm test:watch

# Coverage
pnpm test:cov

# E2E tests
pnpm test:e2e

# Debug tests
pnpm test:debug
```

---

## 🚀 Advanced Features

### 1. Swagger/OpenAPI Documentation

Automatic API documentation at `/api/docs`:

```typescript
// Decorators auto-generate docs
@ApiTags('user')  // Groups endpoints
@ApiAuth({        // Documents auth requirement + response
  summary: 'Get user by ID',
  type: UserDto,
  errorResponses: [401, 404],
})
@Get(':id')
async findUser(@Param('id') id: string) {}
```

Access at: `http://localhost:3000/api/docs`

### 2. Frontend API Code Generation

Generate TypeScript API client automatically:

```bash
# In frontend project
pnpm codegen  # Generates from backend's OpenAPI spec

# Use in React/Vue/etc
import { useGetUserById } from '@/api/queries';

function UserProfile({ userId }) {
  const { data: user } = useGetUserById(userId);
  return <div>{user?.email}</div>;
}
```

### 3. Database Monitoring (Prometheus + Grafana)

Enable in `.env`:
```bash
COMPOSE_PROFILES=monitoring
```

Then access:
- Grafana: `http://localhost:3001`
- Prometheus: `http://localhost:9090`

Pre-built dashboards for:
- API performance metrics
- Database query performance
- Redis cache hit rates
- Queue processing stats

### 4. Dependency Graph Visualization

```bash
# Install Graphviz first
brew install graphviz  # macOS
sudo apt install graphviz  # Linux

# Generate graph
pnpm graph:app  # Full dependency graph
pnpm graph:circular  # Only circular dependencies

# View at .tmp/graph.png
```

### 5. Database ERD Generation

```bash
pnpm erd:generate

# View at .tmp/erd.png
```

### 6. Bull Board (Queue Monitoring)

Access at: `http://localhost:3000/api/queues`

Features:
- View all queues
- Inspect jobs (pending, completed, failed)
- Retry failed jobs
- Job statistics

---

## 💡 Development Tips

### 1. Environment Variables

```bash
# Always use typed config service
❌ process.env.DATABASE_HOST
✅ this.configService.get('database.host', { infer: true })
```

### 2. Database Queries

```typescript
// Use QueryBuilder for complex queries
❌ await this.userRepository.find({ where: { /* complex */ } });

✅ await this.userRepository
     .createQueryBuilder('user')
     .leftJoinAndSelect('user.posts', 'post')
     .where('user.isActive = :active', { active: true })
     .andWhere('post.createdAt > :date', { date: new Date() })
     .getMany();
```

### 3. Error Messages

```typescript
// Always use i18n for user-facing messages
❌ throw new NotFoundException('User not found');
✅ throw new NotFoundException(this.i18nService.t('user.notFound'));
```

### 4. Async/Await

```typescript
// Always await promises in services
❌ this.emailService.send(email);
✅ await this.emailService.send(email);
```

### 5. DTO Transformation

```typescript
// Let class-transformer handle it
export class UserDto {
  @Exclude()  // Never send password
  password: string;

  @Expose()
  get fullName(): string {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

### 6. Logging

```typescript
// Inject logger
constructor(private readonly logger: Logger) {}

// Use appropriate levels
this.logger.log('User created');
this.logger.warn('Unusual activity detected');
this.logger.error('Failed to connect', error.stack);
this.logger.debug('Processing', { data });
```

### 7. Performance

```typescript
// Use cursor pagination for large datasets
❌ offset-based: OFFSET 1000000 LIMIT 10
✅ cursor-based: WHERE createdAt < :cursor LIMIT 10

// Cache expensive queries
const cacheKey = `users:${id}`;
const cached = await this.cacheService.get(cacheKey);
if (cached) return cached;

const user = await this.findUser(id);
await this.cacheService.set(cacheKey, user, 3600);
```

### 8. Security

```typescript
// Always validate and sanitize input
export class CreatePostDto {
  @IsString()
  @MaxLength(255)
  @Transform(({ value }) => value.trim())
  title: string;
}

// Use proper HTTP status codes
❌ return { success: false, error: 'Not found' };
✅ throw new NotFoundException('Resource not found');
```

---

## 📖 Recommended Learning Order

### Week 1: Foundations
1. Set up local environment
2. Study health module and basic endpoints
3. Understand configuration system
4. Learn DTOs and validation

### Week 2: Core Features
1. Deep dive into User module
2. Study authentication flow
3. Practice pagination (offset & cursor)
4. Experiment with file uploads

### Week 3: Advanced Topics
1. Background jobs with BullMQ
2. WebSocket implementation
3. Email templates with React Email
4. Caching strategies

### Week 4: Production Ready
1. Testing strategies
2. Monitoring and logging
3. Database migrations
4. Docker deployment
5. CI/CD with GitHub Actions

---

## 🔗 Additional Resources

### Official Documentation
- [NestJS Docs](https://docs.nestjs.com/)
- [TypeORM Docs](https://typeorm.io/)
- [Better Auth Docs](https://www.better-auth.com/)
- [BullMQ Docs](https://docs.bullmq.io/)

### Related Projects
- [Frontend Client](https://github.com/niraj-khatiwada/ultimate-nestjs-client) - React example
- [Original Boilerplate](https://github.com/vndevteam/nestjs-boilerplate) - Extended from

### Community
- [NestJS Discord](https://discord.gg/nestjs)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/nestjs)

---

## 🎯 Practice Exercises

### Exercise 1: Add a New Entity
Create a "Category" entity with CRUD operations, relationships to other entities, and proper DTOs.

### Exercise 2: Implement a Background Job
Create a job that processes bulk user imports from CSV files.

### Exercise 3: Add Real-time Features
Implement a notification system using WebSockets that updates users in real-time.

### Exercise 4: Extend Authentication
Add social login (Google, GitHub) using Better Auth's OAuth plugins.

### Exercise 5: Performance Optimization
Add Redis caching to a frequently accessed endpoint and measure performance improvement.

---

## ❓ Common Questions

**Q: Why Fastify instead of Express?**
A: Fastify is faster and has better TypeScript support. It's a drop-in replacement with similar API.

**Q: When should I use GraphQL vs REST?**
A: Use REST for simple CRUD, GraphQL when clients need flexible data fetching. This template supports both!

**Q: How do I handle file uploads to S3?**
A: Set `APP_LOCAL_FILE_UPLOAD=false` and configure AWS credentials. The FileService handles the rest.

**Q: Should I use offset or cursor pagination?**
A: Cursor for infinite scroll and large datasets. Offset for traditional page-based UI.

**Q: How do I add a new language?**
A: Add translation files in `src/i18n/translations/{locale}/`, follow existing structure.

---

## 🙏 Contributing

Found something unclear in this guide? Open an issue or PR!

---

**Happy Learning! 🚀**

Remember: The best way to learn is by doing. Start small, build features, make mistakes, and iterate. This boilerplate provides a solid foundation following industry best practices.
