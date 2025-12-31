# 🚀 Quick Reference Card

A quick reference for common tasks and patterns in this NestJS boilerplate.

> 💡 **New here?** Start with [LEARNING_GUIDE.md](./LEARNING_GUIDE.md) for comprehensive understanding.

---

## 📦 Common Commands

### Development
```bash
pnpm start:dev              # Start dev server with hot-reload
pnpm start:debug            # Start with debugger
pnpm docker:dev:up          # Start all Docker services
pnpm docker:dev:down        # Stop Docker services
```

### Database
```bash
pnpm migration:generate src/database/migrations/MigrationName  # Create migration
pnpm migration:up           # Run pending migrations
pnpm migration:down         # Revert last migration
pnpm migration:show         # Show migration status
pnpm seed:run               # Run database seeders
```

### Testing
```bash
pnpm test                   # Run unit tests
pnpm test:watch             # Watch mode
pnpm test:cov               # With coverage
pnpm test:e2e               # E2E tests
```

### Email Development
```bash
pnpm email:dev              # Preview templates at http://localhost:3002
pnpm email:build            # Build templates to .hbs
pnpm email:watch            # Watch and rebuild
```

### Code Quality
```bash
pnpm lint                   # Lint and fix
pnpm format                 # Format with Prettier
```

### Visualization
```bash
pnpm graph:app              # Generate dependency graph
pnpm graph:circular         # Show circular dependencies
pnpm erd:generate           # Generate database ERD
```

### Production
```bash
pnpm build                  # Build for production
pnpm docker:prod:up         # Start production containers
sh ./bin/deploy.sh          # Deploy script
```

---

## 🎯 Common Patterns

### 1. Creating a Controller Endpoint

```typescript
@ApiTags('resource')
@Controller({ path: 'resource', version: '1' })
@UseGuards(AuthGuard)
export class ResourceController {
  constructor(private readonly service: ResourceService) {}

  @Get(':id')
  @ApiAuth({ summary: 'Get resource by ID', type: ResourceDto })
  async findOne(@Param('id', ParseUUIDPipe) id: string) {
    return this.service.findOne(id);
  }

  @Post()
  @ApiAuth({ summary: 'Create resource', type: ResourceDto })
  async create(@Body() dto: CreateResourceDto) {
    return this.service.create(dto);
  }
}
```

### 2. Creating a DTO

```typescript
export class CreateResourceDto {
  @IsString()
  @IsNotEmpty()
  @ApiProperty({ description: 'Resource name' })
  name: string;

  @IsString()
  @IsOptional()
  @ApiPropertyOptional({ description: 'Description' })
  description?: string;

  @IsEmail()
  @ApiProperty({ description: 'Contact email' })
  email: string;
}

export class ResourceDto {
  @ApiProperty()
  id: string;

  @ApiProperty()
  name: string;

  @ApiProperty({ required: false })
  description?: string;

  @ApiProperty()
  @Exclude()  // Never expose in responses
  secretField: string;
}
```

### 3. Creating a Service

```typescript
@Injectable()
export class ResourceService {
  constructor(
    @InjectRepository(ResourceEntity)
    private readonly repository: Repository<ResourceEntity>,
    private readonly i18nService: I18nService,
  ) {}

  async findOne(id: string): Promise<ResourceDto> {
    const resource = await this.repository.findOne({ where: { id } });
    if (!resource) {
      throw new NotFoundException(
        this.i18nService.t('resource.notFound')
      );
    }
    return resource;
  }

  async create(dto: CreateResourceDto): Promise<ResourceDto> {
    const resource = this.repository.create(dto);
    return await this.repository.save(resource);
  }

  async delete(id: string): Promise<void> {
    await this.repository.softDelete(id);
  }
}
```

### 4. Creating an Entity

```typescript
import { BaseModel } from '@/database/models/base.model';

@Entity('resources')
export class ResourceEntity extends BaseModel {
  @Column({ type: 'varchar', length: 255 })
  name: string;

  @Column({ type: 'text', nullable: true })
  description?: string;

  @Column({ type: 'varchar', unique: true })
  email: string;

  @ManyToOne(() => UserEntity, (user) => user.resources)
  @JoinColumn({ name: 'user_id' })
  user: UserEntity;

  @OneToMany(() => CommentEntity, (comment) => comment.resource)
  comments: CommentEntity[];
}
```

### 5. Pagination (Offset)

```typescript
// DTO
export class QueryResourcesDto extends OffsetPaginationDto {}

// Service
async findAll(dto: QueryResourcesDto): Promise<OffsetPaginatedDto<ResourceDto>> {
  const query = this.repository
    .createQueryBuilder('resource')
    .orderBy('resource.createdAt', 'DESC');
  
  const [resources, metaDto] = await paginate<ResourceEntity>(query, dto);
  return new OffsetPaginatedDto(resources, metaDto);
}

// Controller
@Get()
@ApiAuth({ type: OffsetPaginatedResourceDto, isPaginated: true })
async findAll(@Query() dto: QueryResourcesDto) {
  return this.service.findAll(dto);
}
```

### 6. Pagination (Cursor)

```typescript
// DTO
export class QueryResourcesCursorDto extends CursorPaginationDto {}

// Service
async findAllCursor(dto: QueryResourcesCursorDto) {
  const queryBuilder = this.repository.createQueryBuilder('resource');
  const paginator = buildPaginator({
    entity: ResourceEntity,
    alias: 'resource',
    paginationKeys: ['createdAt'],
    query: {
      limit: dto.limit,
      order: 'DESC',
      afterCursor: dto.afterCursor,
      beforeCursor: dto.beforeCursor,
    },
  });

  const { data, cursor } = await paginator.paginate(queryBuilder);
  const metaDto = new CursorPaginationDto(
    data.length,
    cursor.afterCursor,
    cursor.beforeCursor,
    dto,
  );

  return new CursorPaginatedDto(data, metaDto);
}
```

### 7. Background Jobs

```typescript
// Producer (enqueue job)
@Injectable()
export class NotificationService {
  constructor(
    @InjectQueue(Queue.Notification) 
    private queue: Queue
  ) {}

  async sendEmail(userId: string, message: string) {
    await this.queue.add('send-email', { userId, message }, {
      attempts: 3,
      backoff: { type: 'exponential', delay: 2000 },
    });
  }
}

// Consumer (process job)
@Processor(Queue.Notification)
export class NotificationProcessor {
  @Process('send-email')
  async handleEmail(job: Job<{ userId: string; message: string }>) {
    const { userId, message } = job.data;
    // Send email logic
    await this.emailService.send(userId, message);
  }
}
```

### 8. Caching

```typescript
@Injectable()
export class ResourceService {
  constructor(
    private readonly cacheService: CacheService,
    @InjectRepository(ResourceEntity)
    private readonly repository: Repository<ResourceEntity>,
  ) {}

  async findOne(id: string): Promise<ResourceDto> {
    const cacheKey = `resource:${id}`;
    
    // Try cache first
    const cached = await this.cacheService.get<ResourceDto>(cacheKey);
    if (cached) return cached;

    // Fetch from DB
    const resource = await this.repository.findOne({ where: { id } });
    if (!resource) {
      throw new NotFoundException('Resource not found');
    }

    // Cache for 1 hour
    await this.cacheService.set(cacheKey, resource, 3600);
    
    return resource;
  }

  async update(id: string, dto: UpdateResourceDto) {
    const result = await this.repository.update(id, dto);
    
    // Invalidate cache
    await this.cacheService.del(`resource:${id}`);
    
    return result;
  }
}
```

### 9. Custom Decorators

```typescript
// Getting current user
@Get('profile')
async getProfile(
  @CurrentUserSession('user') user: CurrentUserSession['user']
) {
  return user;
}

// Full session
@Get('session')
async getSession(@CurrentUserSession() session: CurrentUserSession) {
  return session;
}

// Optional authentication
@OptionalAuth()
@Get('public-or-private')
async flexible(@CurrentUserSession() session?: CurrentUserSession) {
  if (session) {
    return 'Private content';
  }
  return 'Public content';
}
```

### 10. File Upload

```typescript
// Local storage
@Post('upload')
@UseInterceptors(FileInterceptor('file'))
@ApiAuth({ summary: 'Upload file' })
async uploadFile(@UploadedFile() file: Express.Multer.File) {
  return this.fileService.upload(file);
}

// Multiple files
@Post('upload-multiple')
@UseInterceptors(FilesInterceptor('files', 10))
async uploadFiles(@UploadedFiles() files: Express.Multer.File[]) {
  return Promise.all(files.map(f => this.fileService.upload(f)));
}
```

### 11. GraphQL Resolver

```typescript
@Resolver(() => ResourceSchema)
export class ResourceResolver {
  constructor(private readonly service: ResourceService) {}

  @Query(() => ResourceSchema, { name: 'resource' })
  async getResource(@Args('id') id: string) {
    return this.service.findOne(id);
  }

  @Query(() => [ResourceSchema], { name: 'resources' })
  async getResources() {
    return this.service.findAll({});
  }

  @Mutation(() => ResourceSchema)
  async createResource(@Args('input') input: CreateResourceInput) {
    return this.service.create(input);
  }

  @Mutation(() => Boolean)
  async deleteResource(@Args('id') id: string) {
    await this.service.delete(id);
    return true;
  }

  // Field resolver
  @ResolveField(() => UserSchema)
  async user(@Parent() resource: ResourceEntity) {
    return this.userService.findOne(resource.userId);
  }
}
```

### 12. WebSocket Events

```typescript
@WebSocketGateway({ cors: { origin: '*' } })
export class EventsGateway {
  @WebSocketServer()
  server: Server;

  @SubscribeMessage('message')
  handleMessage(
    @MessageBody() data: string,
    @ConnectedSocket() client: Socket,
  ): void {
    // Emit to all clients
    this.server.emit('message', data);
    
    // Emit to sender only
    client.emit('message', data);
    
    // Emit to room
    this.server.to('room-name').emit('message', data);
  }

  @SubscribeMessage('join-room')
  handleJoinRoom(
    @MessageBody() room: string,
    @ConnectedSocket() client: Socket,
  ) {
    client.join(room);
    return { event: 'joined', room };
  }
}
```

---

## 🗂️ File Locations

| What | Where |
|------|-------|
| API Endpoints | `src/api/{feature}/{feature}.controller.ts` |
| Business Logic | `src/api/{feature}/{feature}.service.ts` |
| Database Entities | `src/auth/entities/` or `src/api/{feature}/entities/` |
| DTOs | `src/api/{feature}/dto/` or `src/common/dto/` |
| GraphQL Schemas | `src/api/{feature}/schema/` |
| Custom Decorators | `src/decorators/` |
| Configurations | `src/config/{service}/` |
| Migrations | `src/database/migrations/` |
| Seeds | `src/database/seeds/` |
| Email Templates | `src/shared/mail/templates/` |
| Translations | `src/i18n/translations/{locale}/` |
| Background Jobs | `src/worker/queues/{queue}/` |
| Utils | `src/utils/` |
| Tests (unit) | `*.spec.ts` next to the file |
| Tests (e2e) | `test/*.e2e-spec.ts` |

---

## 🔑 Environment Variables

### Essential Variables

```bash
# App
NODE_ENV=development
APP_NAME=my-app
APP_PORT=3000
APP_WORKER_PORT=3001
APP_URL=http://localhost:3000
APP_CORS_ORIGIN=http://localhost:5173

# Database
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USERNAME=postgres
DATABASE_PASSWORD=postgres
DATABASE_NAME=myapp
DATABASE_SYNCHRONIZE=false

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# Auth
AUTH_SECRET=your-secret-key-here
AUTH_BASE_PATH=/api/auth
AUTH_TRUST_HOST=true

# Email
MAIL_HOST=localhost
MAIL_PORT=1025
MAIL_FROM_EMAIL=noreply@example.com
MAIL_FROM_NAME=MyApp

# AWS (optional)
APP_LOCAL_FILE_UPLOAD=true
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_S3_BUCKET=
AWS_S3_REGION=

# Monitoring (optional)
COMPOSE_PROFILES=monitoring  # Enable Prometheus/Grafana
```

---

## 🚨 Common Errors & Solutions

### 1. Migration Errors
```bash
# Error: relation does not exist
# Solution: Run pending migrations
pnpm migration:up

# Error: Cannot generate migration
# Solution: Build first, then generate
pnpm build && pnpm migration:generate src/database/migrations/Name
```

### 2. Port Already in Use
```bash
# Find and kill process
lsof -ti:3000 | xargs kill -9
# Or change port in .env
APP_PORT=3001
```

### 3. Docker Issues
```bash
# Reset everything
pnpm docker:dev:down
docker system prune -a
pnpm docker:dev:up

# View logs
docker logs nestjs-boilerplate-server
```

### 4. Type Errors
```typescript
// Error: Type inference not working
// Solution: Use infer option
const value = this.configService.get('app.port', { infer: true });

// Error: Circular dependency
// Solution: Use forwardRef
@Module({
  imports: [forwardRef(() => OtherModule)],
})
```

---

## 📚 URLs During Development

| Service | URL |
|---------|-----|
| API Server | http://localhost:3000 |
| Worker Server | http://localhost:3001 |
| Swagger Docs | http://localhost:3000/api/docs |
| Better Auth Reference | http://localhost:3000/api/auth/reference |
| Bull Board (Queues) | http://localhost:3000/api/queues |
| GraphQL Playground | http://localhost:3000/graphql |
| React Email Preview | http://localhost:3002 |
| MailPit (Email Testing) | http://localhost:18025 |
| Grafana (Monitoring) | http://localhost:3001 (if enabled) |
| Prometheus | http://localhost:9090 (if enabled) |

---

## 💡 Pro Tips

1. **Use TypeORM QueryBuilder** for complex queries instead of find options
2. **Always use i18n** for user-facing error messages
3. **Cache expensive operations** with Redis
4. **Use cursor pagination** for large datasets
5. **Validate environment variables** early in the config files
6. **Soft delete** instead of hard delete (data retention)
7. **Use Bull Board** to debug queue issues visually
8. **Test emails** with MailPit instead of real SMTP in dev
9. **Monitor with Grafana** when performance testing
10. **Generate API client** for frontend using OpenAPI codegen

---

## 🔗 Quick Links

- [Learning Guide](./LEARNING_GUIDE.md) - Comprehensive guide
- [README](./README.md) - Project overview
- [NestJS Docs](https://docs.nestjs.com/)
- [TypeORM Docs](https://typeorm.io/)
- [Better Auth Docs](https://www.better-auth.com/)
- [Frontend Example](https://github.com/niraj-khatiwada/ultimate-nestjs-client)

---

**Need help?** Check the [LEARNING_GUIDE.md](./LEARNING_GUIDE.md) for detailed explanations! 🚀
