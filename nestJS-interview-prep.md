# NestJS Interview Prep – Top 5 Most Common Questions

These are very common questions for Senior Full Stack, Node.js, and NestJS interviews.

⸻

## 1. What is NestJS and why would you use it?

Answer

NestJS is a progressive Node.js framework built on top of Express (or Fastify) that uses TypeScript and follows Angular-style architecture.

Key Benefits

-- Dependency Injection
-- Modular Architecture
-- TypeScript First
-- Built-in Testing
-- Microservices Support
-- WebSocket Support
-- GraphQL Support

Example
```ts
@Controller('customers')
export class CustomerController {
  @Get()
  findAll() {
    return ['John', 'Mary'];
  }
}
```
Request:

GET /customers

Response:
```ts
[
  "John",
  "Mary"
]
```
Interview Talking Point

NestJS provides enterprise-level architecture similar to Spring Boot. It encourages clean separation between controllers, services, and modules.

⸻

## 2. Explain Dependency Injection in NestJS

Answer

Dependency Injection (DI) allows NestJS to automatically create and inject dependencies.

Without DI:
```ts
const service =
    new CustomerService();
```
With DI:

### Service
```ts
@Injectable()
export class CustomerService {
  findAll() {
    return ['John', 'Mary'];
  }
}
```
### Controller
```ts
@Controller('customers')
export class CustomerController {
  constructor(
    private readonly customerService: CustomerService
  ) {}
  @Get()
  findAll() {
    return this.customerService.findAll();
  }
}
```
### Module
```ts
@Module({
  controllers: [CustomerController],
  providers: [CustomerService]
})
export class CustomerModule {}
```
Interview Talking Point

NestJS Dependency Injection is heavily inspired by Angular and Spring Framework.

⸻

## 3. What are Modules in NestJS?

Answer

Modules organize related functionality.

Think of them like:

-- Customer Module
-- Order Module
-- Payment Module
-- Notification Module

Example
```ts
@Module({
  controllers: [CustomerController],
  providers: [CustomerService]
})
export class CustomerModule {}
```
Root Module
```ts
@Module({
  imports: [
    CustomerModule,
    OrderModule
  ]
})
export class AppModule {}
```
### Benefits

-- Separation of concerns
-- Scalability
-- Maintainability

Interview Talking Point

Large enterprise systems may contain dozens of modules.

⸻

## 4. What is the difference between Controller and Service?

### ontroller

Handles HTTP requests.
```ts
@Controller('customers')
export class CustomerController {
  @Get()
  findAll() {
    return this.customerService.findAll();
  }
}
```
Responsibilities

-- Receive requests
-- Validate parameters
-- Return responses

⸻

### Service

Contains business logic.
```ts
@Injectable()
export class CustomerService {
  findAll() {
    return ['John', 'Mary'];
  }
}
```
### Responsibilities

-- Business rules
-- Database calls
-- External API calls
-- Calculations

⸻

Interview Talking Point

Controllers should remain thin.
Services should contain business logic.

Bad:
```ts
@Get()
findAll() {
  // huge business logic
}
```
Good:

@Get()
findAll() {
  return this.customerService.findAll();
}

⸻

## 5. What are Guards in NestJS?

Answer

Guards determine whether a request is allowed to proceed.

Think:

-- Authentication
-- Authorization
-- Role Validation
-- Permissions

⸻

Example Guard
```ts
@Injectable()
export class AuthGuard
  implements CanActivate {
  canActivate(
    context: ExecutionContext
  ): boolean {
    const request =
      context.switchToHttp()
             .getRequest();
    return !!request.headers.authorization;
  }
}
```
⸻

Apply Guard
```ts
@UseGuards(AuthGuard)
@Get()
findAll() {
  return [];
}
```
⸻

JWT Example
```ts
@UseGuards(JwtAuthGuard)
@Get('profile')
profile() {
   return 'Secure Data';
}
```
⸻

Request Flow
```tx
Request
   ↓
Guard
   ↓
Controller
   ↓
Service
   ↓
Response
```
⸻

Interview Talking Point

Guards are equivalent to Spring Security filters and are commonly used with:

-- JWT
-- OAuth
-- Roles
-- Permissions
-- RBAC

⸻

## Senior Follow-Up Questions

Be prepared for:

1. What are Interceptors?
2. What are Middleware?
3. What are Pipes?
4. How does Exception Handling work?
5. How do you implement JWT Authentication?
6. How do you connect NestJS to MongoDB?
7. How do you connect NestJS to PostgreSQL?
8. Explain TypeORM vs Prisma.
9. Explain Microservices in NestJS.
10. How would you secure a NestJS REST API?

These are the next 5 most common NestJS interview questions and are frequently asked for Senior Full Stack and Node.js positions.