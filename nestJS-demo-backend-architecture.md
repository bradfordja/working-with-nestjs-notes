````md
# NestJS Demo Backend Architecture

## Demo Use Case

A small backend for:

- UI Client → NestJS API Gateway
- API Gateway → Internal Orders Microservice
- PostgreSQL database
- JWT security for frontend users
- API-key security for API-to-API calls
- Kafka for async events
- AWS-ready deployment design

---

# 1. Recommended Architecture

```txt
React / Angular / Next.js UI
        |
        | JWT Bearer Token
        v
NestJS API Gateway
        |
        | Internal API Key / Private Network
        v
Orders Microservice
        |
        v
PostgreSQL / AWS RDS

Orders Microservice
        |
        | Kafka Event: order.created
        v
Kafka / MSK
        |
        v
Notification / Payment / Audit Services
````

---

# 2. Create NestJS Apps

```bash
npm i -g @nestjs/cli

nest new api-gateway
nest new orders-service
```

Install common packages:

```bash
npm install @nestjs/config @nestjs/jwt @nestjs/passport passport passport-jwt
npm install @nestjs/typeorm typeorm pg
npm install class-validator class-transformer
npm install @nestjs/microservices kafkajs
```

---

# 3. API Gateway `.env`

```env
PORT=3000
JWT_SECRET=super-secret-key
ORDERS_SERVICE_URL=http://localhost:3001
INTERNAL_API_KEY=internal-api-secret
```

---

# 4. Orders Service `.env`

```env
PORT=3001

DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USER=postgres
DATABASE_PASSWORD=postgres
DATABASE_NAME=demo_orders

INTERNAL_API_KEY=internal-api-secret

KAFKA_BROKER=localhost:9092
```

---

# 5. API Gateway Main App

## `main.ts`

```ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import helmet from 'helmet';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.use(helmet());

  app.enableCors({
    origin: ['http://localhost:3000', 'http://localhost:4200'],
    credentials: true,
  });

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
    }),
  );

  await app.listen(process.env.PORT || 3000);
}
bootstrap();
```

---

# 6. API Gateway Auth Module

## `auth.service.ts`

```ts
import { Injectable } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class AuthService {
  constructor(private readonly jwtService: JwtService) {}

  login() {
    const user = {
      id: 1,
      email: 'admin@test.com',
      role: 'admin',
    };

    return {
      accessToken: this.jwtService.sign(user),
    };
  }
}
```

## `auth.controller.ts`

```ts
import { Controller, Post } from '@nestjs/common';
import { AuthService } from './auth.service';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Post('login')
  login() {
    return this.authService.login();
  }
}
```

---

# 7. JWT Guard for UI Client Security

## `jwt-auth.guard.ts`

```ts
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(private readonly jwtService: JwtService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();

    const authHeader = request.headers.authorization;

    if (!authHeader) {
      throw new UnauthorizedException('Missing Authorization header');
    }

    const token = authHeader.replace('Bearer ', '');

    try {
      const payload = this.jwtService.verify(token);
      request.user = payload;
      return true;
    } catch {
      throw new UnauthorizedException('Invalid JWT token');
    }
  }
}
```

---

# 8. API Gateway Orders Controller

## `orders.controller.ts`

```ts
import { Body, Controller, Get, Post, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../security/jwt-auth.guard';
import { OrdersGatewayService } from './orders-gateway.service';

@Controller('orders')
@UseGuards(JwtAuthGuard)
export class OrdersController {
  constructor(private readonly ordersGatewayService: OrdersGatewayService) {}

  @Post()
  create(@Body() body: any) {
    return this.ordersGatewayService.createOrder(body);
  }

  @Get()
  findAll() {
    return this.ordersGatewayService.findAll();
  }
}
```

## `orders-gateway.service.ts`

```ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class OrdersGatewayService {
  private readonly ordersUrl = process.env.ORDERS_SERVICE_URL;
  private readonly apiKey = process.env.INTERNAL_API_KEY;

  async createOrder(order: any) {
    const response = await fetch(`${this.ordersUrl}/internal/orders`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'x-api-key': this.apiKey,
      },
      body: JSON.stringify(order),
    });

    return response.json();
  }

  async findAll() {
    const response = await fetch(`${this.ordersUrl}/internal/orders`, {
      headers: {
        'x-api-key': this.apiKey,
      },
    });

    return response.json();
  }
}
```

---

# 9. Orders Service Database Connection

## `app.module.ts`

```ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { OrdersModule } from './orders/orders.module';
import { Order } from './orders/order.entity';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: process.env.DATABASE_HOST,
      port: Number(process.env.DATABASE_PORT),
      username: process.env.DATABASE_USER,
      password: process.env.DATABASE_PASSWORD,
      database: process.env.DATABASE_NAME,
      entities: [Order],
      synchronize: true,
    }),
    OrdersModule,
  ],
})
export class AppModule {}
```

---

# 10. Order Entity

## `order.entity.ts`

```ts
import {
  Column,
  Entity,
  PrimaryGeneratedColumn,
  CreateDateColumn,
} from 'typeorm';

@Entity('orders')
export class Order {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  customerName: string;

  @Column('decimal')
  amount: number;

  @Column({ default: 'PENDING' })
  status: string;

  @CreateDateColumn()
  createdAt: Date;
}
```

---

# 11. API-to-API Security Guard

## `internal-api-key.guard.ts`

```ts
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';

@Injectable()
export class InternalApiKeyGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();

    const apiKey = request.headers['x-api-key'];

    if (apiKey !== process.env.INTERNAL_API_KEY) {
      throw new UnauthorizedException('Invalid internal API key');
    }

    return true;
  }
}
```

---

# 12. Orders Service Controller

## `orders.controller.ts`

```ts
import { Body, Controller, Get, Post, UseGuards } from '@nestjs/common';
import { InternalApiKeyGuard } from '../security/internal-api-key.guard';
import { OrdersService } from './orders.service';

@Controller('internal/orders')
@UseGuards(InternalApiKeyGuard)
export class OrdersController {
  constructor(private readonly ordersService: OrdersService) {}

  @Post()
  create(@Body() body: any) {
    return this.ordersService.create(body);
  }

  @Get()
  findAll() {
    return this.ordersService.findAll();
  }
}
```

---

# 13. Orders Service with Kafka Event

## `orders.service.ts`

```ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { ClientKafka } from '@nestjs/microservices';
import { Repository } from 'typeorm';
import { Order } from './order.entity';

@Injectable()
export class OrdersService {
  constructor(
    @InjectRepository(Order)
    private readonly orderRepository: Repository<Order>,

    private readonly kafkaClient: ClientKafka,
  ) {}

  async create(data: Partial<Order>) {
    const order = this.orderRepository.create(data);

    const savedOrder = await this.orderRepository.save(order);

    this.kafkaClient.emit('order.created', {
      id: savedOrder.id,
      customerName: savedOrder.customerName,
      amount: savedOrder.amount,
      status: savedOrder.status,
    });

    return savedOrder;
  }

  findAll() {
    return this.orderRepository.find();
  }
}
```

---

# 14. Kafka Client Configuration

## `orders.module.ts`

```ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ClientsModule, Transport } from '@nestjs/microservices';
import { Order } from './order.entity';
import { OrdersController } from './orders.controller';
import { OrdersService } from './orders.service';

@Module({
  imports: [
    TypeOrmModule.forFeature([Order]),

    ClientsModule.register([
      {
        name: 'KAFKA_CLIENT',
        transport: Transport.KAFKA,
        options: {
          client: {
            brokers: [process.env.KAFKA_BROKER || 'localhost:9092'],
          },
          producerOnlyMode: true,
        },
      },
    ]),
  ],
  controllers: [OrdersController],
  providers: [
    OrdersService,
    {
      provide: 'ClientKafka',
      useExisting: 'KAFKA_CLIENT',
    },
  ],
})
export class OrdersModule {}
```

A cleaner version would inject this directly:

```ts
@Inject('KAFKA_CLIENT')
private readonly kafkaClient: ClientKafka
```

---

# 15. Kafka Consumer Example

## Notification Service

```ts
import { Controller } from '@nestjs/common';
import { EventPattern, Payload } from '@nestjs/microservices';

@Controller()
export class NotificationConsumer {
  @EventPattern('order.created')
  handleOrderCreated(@Payload() event: any) {
    console.log('Send notification for order:', event);
  }
}
```

---

# 16. Docker Compose for Local Development

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16
    container_name: demo-postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: demo_orders
    ports:
      - "5432:5432"

  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

---

# 17. AWS Recommended Microservice Configuration

## Production AWS Design

```txt
Frontend:
- S3 + CloudFront
- Or Amplify for React/Angular/Next.js hosting

API Layer:
- API Gateway or ALB
- NestJS API Gateway running on ECS Fargate or EKS

Microservices:
- Orders Service on ECS Fargate or EKS
- Notification Service on ECS Fargate or EKS
- Payment Service on ECS Fargate or EKS

Database:
- Amazon RDS PostgreSQL
- Secrets Manager for credentials

Messaging:
- Amazon MSK for Kafka
- Or Amazon SQS/SNS for simpler async processing

Security:
- Cognito for user login
- JWT validation in NestJS
- Internal APIs private inside VPC
- Security Groups between services
- Secrets Manager for API keys
- IAM roles for AWS access
- AWS WAF for edge protection

Monitoring:
- CloudWatch Logs
- X-Ray tracing
- OpenTelemetry
- Prometheus/Grafana if using EKS
```

---

# 18. Recommended Security Design

## UI Client to API Gateway

Use:

```txt
Authorization: Bearer <JWT>
```

Best choice:

```txt
AWS Cognito + JWT + NestJS Guard
```

## API Gateway to Internal Microservices

For production, prefer:

```txt
Private VPC networking
Security Groups
Internal ALB
API key or signed service token
mTLS if high-security environment
```

## API Gateway to AWS Services

Use:

```txt
IAM Role for Task / Pod
Secrets Manager
Parameter Store
```

---

# 19. Best Practice Folder Structure

```txt
src/
  app.module.ts
  main.ts

  auth/
    auth.controller.ts
    auth.service.ts
    jwt-auth.guard.ts

  orders/
    orders.controller.ts
    orders-gateway.service.ts
    dto/
      create-order.dto.ts

  common/
    guards/
    filters/
    interceptors/

  config/
    app.config.ts
```

For the Orders Service:

```txt
src/
  app.module.ts
  main.ts

  orders/
    orders.controller.ts
    orders.service.ts
    order.entity.ts
    dto/
      create-order.dto.ts

  security/
    internal-api-key.guard.ts

  kafka/
    kafka.module.ts
```

---

# 20. Senior Interview Explanation

## Question

How would you secure communication between a UI client, NestJS API Gateway, internal microservices, database, and Kafka?

## Strong Senior Answer

For UI-to-backend communication, I would use OAuth2 or Cognito-based login and validate JWT tokens in the NestJS API Gateway using guards.

For API-to-API communication, I would avoid exposing internal services publicly. Internal services should run inside a private VPC, behind an internal load balancer, using security groups to restrict traffic. I may also use API keys, signed service tokens, or mTLS depending on the security requirements.

For database access, I would use RDS PostgreSQL in private subnets. Credentials should be stored in AWS Secrets Manager, not environment files. Services should connect using least-privilege credentials.

For Kafka, I would use Amazon MSK with private networking, TLS encryption, IAM authentication if supported, and topic-level access control.

For observability, I would use CloudWatch, structured logging, correlation IDs, distributed tracing, and alerting.

The important design principle is that the public client only talks to the API Gateway. Internal services, databases, and Kafka should never be directly exposed to the public internet.

```
```
