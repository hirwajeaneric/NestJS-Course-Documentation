# Module 7: APIs & Integration

## 📚 Table of Contents

1. [Overview](#overview)
2. [RESTful API Design](#restful-api-design)
3. [API Documentation with Swagger](#api-documentation-with-swagger)
4. [GraphQL Integration](#graphql-integration)
5. [WebSocket Implementation](#websocket-implementation)
6. [Rate Limiting](#rate-limiting)
7. [API Versioning](#api-versioning)
8. [CORS Configuration](#cors-configuration)
9. [Best Practices](#best-practices)
10. [Mini Project: Real-time Chat Application](#mini-project-real-time-chat-application)

---

## Overview

This module covers building different types of APIs and integrations in NestJS. You'll learn RESTful API design principles, GraphQL for flexible queries, WebSockets for real-time communication, and how to document and secure your APIs.

**API Types Covered:**

- **REST APIs**: Traditional HTTP-based APIs
- **GraphQL**: Query language for APIs with flexible data fetching
- **WebSocket**: Real-time bidirectional communication
- **API Documentation**: Swagger/OpenAPI for API discovery

---

## RESTful API Design

### REST Principles

REST (Representational State Transfer) is an architectural style for designing networked applications.

**Key REST Principles:**

1. **Stateless**: Each request contains all information needed
2. **Resource-Based**: URLs represent resources, not actions
3. **HTTP Methods**: Use proper HTTP verbs (GET, POST, PUT, PATCH, DELETE)
4. **Status Codes**: Return appropriate HTTP status codes
5. **JSON**: Use JSON for data exchange

### RESTful Routes Structure

Here's how to structure RESTful routes following best practices:

```typescript
// src/users/users.controller.ts
import {
  Controller,
  Get,
  Post,
  Put,
  Patch,
  Delete,
  Body,
  Param,
  Query,
  HttpCode,
  HttpStatus,
} from "@nestjs/common";

// @Controller('users') - Base path for all routes
// RESTful convention: Use plural nouns for resources
@Controller("users")
export class UsersController {
  // GET /users - Get all users (with pagination and filtering)
  // This is a read operation that doesn't modify data
  @Get()
  findAll(
    @Query("page") page: number = 1,
    @Query("limit") limit: number = 10,
    @Query("search") search?: string
  ) {
    // Implementation returns list of users
    return this.usersService.findAll({ page, limit, search });
  }

  // GET /users/:id - Get a single user by ID
  // Path parameter :id represents the resource identifier
  @Get(":id")
  findOne(@Param("id") id: string) {
    // Implementation returns single user
    return this.usersService.findOne(+id);
  }

  // POST /users - Create a new user
  // POST is used for creating new resources
  // Returns 201 Created status code
  @Post()
  @HttpCode(HttpStatus.CREATED)
  create(@Body() createUserDto: CreateUserDto) {
    // Implementation creates new user
    return this.usersService.create(createUserDto);
  }

  // PUT /users/:id - Replace entire user (full update)
  // PUT requires sending all fields, even unchanged ones
  @Put(":id")
  update(@Param("id") id: string, @Body() updateUserDto: UpdateUserDto) {
    // Implementation updates entire user resource
    return this.usersService.update(+id, updateUserDto);
  }

  // PATCH /users/:id - Partially update user
  // PATCH allows updating only specified fields
  @Patch(":id")
  updatePartial(
    @Param("id") id: string,
    @Body() partialUpdateDto: Partial<UpdateUserDto>
  ) {
    // Implementation updates only provided fields
    return this.usersService.updatePartial(+id, partialUpdateDto);
  }

  // DELETE /users/:id - Delete a user
  // Returns 204 No Content on success
  @Delete(":id")
  @HttpCode(HttpStatus.NO_CONTENT)
  remove(@Param("id") id: string) {
    // Implementation deletes user
    return this.usersService.remove(+id);
  }
}
```

**RESTful Route Conventions:**

- Use nouns, not verbs: `/users` not `/getUsers`
- Use plural nouns: `/users` not `/user`
- Use HTTP methods to indicate actions
- Use status codes to indicate results
- Use path parameters for resource IDs
- Use query parameters for filtering, pagination, search

### Nested Resources

For related resources, use nested routes:

```typescript
// src/posts/posts.controller.ts
@Controller("users/:userId/posts")
export class PostsController {
  // GET /users/:userId/posts - Get all posts for a user
  @Get()
  findAll(@Param("userId") userId: string) {
    return this.postsService.findAllByUser(+userId);
  }

  // GET /users/:userId/posts/:postId - Get a specific post
  @Get(":postId")
  findOne(@Param("userId") userId: string, @Param("postId") postId: string) {
    return this.postsService.findOneByUser(+userId, +postId);
  }
}
```

---

## API Documentation with Swagger

Swagger (OpenAPI) automatically generates interactive API documentation from your code.

### Install and Setup

```bash
npm install @nestjs/swagger swagger-ui-express
```

### Configure Swagger

```typescript
// src/main.ts
import { NestFactory } from "@nestjs/core";
import { SwaggerModule, DocumentBuilder } from "@nestjs/swagger";
import { AppModule } from "./app.module";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Create Swagger configuration
  const config = new DocumentBuilder()
    // API title shown in Swagger UI
    .setTitle("E-Commerce API")
    // API description
    .setDescription("Complete API documentation for the E-Commerce Platform")
    // API version
    .setVersion("1.0")
    // Add bearer token authentication
    .addBearerAuth()
    // Add tags for grouping endpoints
    .addTag("users", "User management endpoints")
    .addTag("products", "Product management endpoints")
    .addTag("orders", "Order management endpoints")
    // Build the configuration
    .build();

  // Create Swagger document
  const document = SwaggerModule.createDocument(app, config);

  // Setup Swagger UI endpoint
  // Access at: http://localhost:3000/api/docs
  SwaggerModule.setup("api/docs", app, document, {
    // Customize Swagger UI
    customSiteTitle: "E-Commerce API Docs",
    customCss: ".swagger-ui .topbar { display: none }",
  });

  await app.listen(3000);
}
bootstrap();
```

### Documenting DTOs

Add Swagger decorators to DTOs for automatic documentation:

```typescript
// src/users/dto/create-user.dto.ts
import { ApiProperty } from "@nestjs/swagger";
import { IsEmail, IsString, MinLength, MaxLength } from "class-validator";

export class CreateUserDto {
  // @ApiProperty() documents the property in Swagger
  // It provides description, example, and validation rules
  @ApiProperty({
    description: "User email address",
    example: "user@example.com",
    format: "email",
  })
  @IsEmail()
  email: string;

  @ApiProperty({
    description: "User password",
    example: "SecurePassword123!",
    minLength: 8,
    maxLength: 50,
  })
  @IsString()
  @MinLength(8)
  @MaxLength(50)
  password: string;

  @ApiProperty({
    description: "User first name",
    example: "John",
    required: false, // Optional field
  })
  @IsString()
  firstName?: string;

  @ApiProperty({
    description: "User last name",
    example: "Doe",
    required: false,
  })
  @IsString()
  lastName?: string;
}
```

### Documenting Controllers

Add decorators to controllers and endpoints:

```typescript
// src/users/users.controller.ts
import { Controller, Get, Post, Body, Param } from "@nestjs/common";
import {
  ApiTags,
  ApiOperation,
  ApiResponse,
  ApiParam,
  ApiBearerAuth,
} from "@nestjs/swagger";

// @ApiTags() groups related endpoints in Swagger UI
@ApiTags("users")
@Controller("users")
export class UsersController {
  // @ApiOperation() describes what the endpoint does
  @ApiOperation({
    summary: "Get all users",
    description: "Retrieve a list of all users with pagination",
  })
  // @ApiResponse() documents possible responses
  @ApiResponse({ status: 200, description: "Successfully retrieved users" })
  @ApiResponse({ status: 401, description: "Unauthorized" })
  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  @ApiOperation({ summary: "Get user by ID" })
  // @ApiParam() documents path parameters
  @ApiParam({ name: "id", description: "User ID", type: "number" })
  @ApiResponse({ status: 200, description: "User found" })
  @ApiResponse({ status: 404, description: "User not found" })
  @Get(":id")
  findOne(@Param("id") id: string) {
    return this.usersService.findOne(+id);
  }

  @ApiOperation({ summary: "Create a new user" })
  @ApiResponse({ status: 201, description: "User created successfully" })
  @ApiResponse({ status: 400, description: "Bad request - validation failed" })
  // @ApiBearerAuth() indicates this endpoint requires authentication
  @ApiBearerAuth()
  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }
}
```

---

## GraphQL Integration

GraphQL is a query language that allows clients to request exactly the data they need.

### Install GraphQL

```bash
npm install @nestjs/graphql @nestjs/apollo graphql apollo-server-express
```

### GraphQL Module Setup

```typescript
// src/app.module.ts
import { Module } from "@nestjs/common";
import { GraphQLModule } from "@nestjs/graphql";
import { ApolloDriver, ApolloDriverConfig } from "@nestjs/apollo";
import { join } from "path";

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      // ApolloDriver is the GraphQL server implementation
      driver: ApolloDriver,

      // autoSchemaFile: Automatically generate GraphQL schema from decorators
      // The schema will be written to this file
      autoSchemaFile: join(process.cwd(), "src/schema.gql"),

      // sortSchema: Alphabetically sort schema fields
      sortSchema: true,

      // playground: Enable GraphQL Playground (development only)
      // Access at: http://localhost:3000/graphql
      playground: true,

      // introspection: Allow schema introspection (disabled in production)
      introspection: true,
    }),
  ],
})
export class AppModule {}
```

### GraphQL Resolvers

Resolvers handle GraphQL queries and mutations:

```typescript
// src/users/users.resolver.ts
import { Resolver, Query, Mutation, Args, ID } from "@nestjs/graphql";
import { UsersService } from "./users.service";
import { User } from "./entities/user.entity";
import { CreateUserInput } from "./dto/create-user.input";
import { UpdateUserInput } from "./dto/update-user.input";

// @Resolver() marks this class as a GraphQL resolver
// 'User' is the type this resolver handles
@Resolver(() => User)
export class UsersResolver {
  constructor(private readonly usersService: UsersService) {}

  // @Query() defines a GraphQL query (read operation)
  // Returns an array of User objects
  @Query(() => [User], { name: "users" })
  // @Args() defines query arguments
  findAll(
    @Args("limit", { type: () => Number, nullable: true }) limit?: number
  ) {
    return this.usersService.findAll(limit);
  }

  // Query with argument
  @Query(() => User, { name: "user" })
  findOne(@Args("id", { type: () => ID }) id: string) {
    return this.usersService.findOne(+id);
  }

  // @Mutation() defines a GraphQL mutation (write operation)
  @Mutation(() => User)
  createUser(@Args("createUserInput") createUserInput: CreateUserInput) {
    return this.usersService.create(createUserInput);
  }

  @Mutation(() => User)
  updateUser(
    @Args("id", { type: () => ID }) id: string,
    @Args("updateUserInput") updateUserInput: UpdateUserInput
  ) {
    return this.usersService.update(+id, updateUserInput);
  }

  @Mutation(() => Boolean)
  removeUser(@Args("id", { type: () => ID }) id: string) {
    return this.usersService.remove(+id);
  }
}
```

### GraphQL Entities

Define GraphQL object types:

```typescript
// src/users/entities/user.entity.ts
import { ObjectType, Field, Int, ID } from "@nestjs/graphql";

// @ObjectType() defines a GraphQL object type
@ObjectType()
export class User {
  // @Field() exposes the property in GraphQL schema
  // () => ID means this field is of type ID
  @Field(() => ID)
  id: number;

  @Field()
  email: string;

  @Field({ nullable: true }) // nullable: true means field can be null
  firstName?: string;

  @Field({ nullable: true })
  lastName?: string;

  // password is NOT exposed in GraphQL (security)
  // Fields without @Field() are not in the schema
  password: string;

  @Field(() => Int)
  createdAt: Date;
}
```

---

## WebSocket Implementation

WebSockets enable real-time bidirectional communication between client and server.

### Install WebSocket

```bash
npm install @nestjs/websockets @nestjs/platform-socket.io socket.io
```

### WebSocket Gateway

Gateways handle WebSocket connections:

```typescript
// src/chat/chat.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  OnGatewayConnection,
  OnGatewayDisconnect,
  MessageBody,
  ConnectedSocket,
} from "@nestjs/websockets";
import { Server, Socket } from "socket.io";

// @WebSocketGateway() creates a WebSocket server
// namespace: '/chat' creates a namespace (optional)
// cors: { origin: '*' } allows all origins (configure properly in production)
@WebSocketGateway({
  namespace: "/chat",
  cors: { origin: "*" },
})
export class ChatGateway implements OnGatewayConnection, OnGatewayDisconnect {
  // @WebSocketServer() gives access to the Socket.IO server instance
  @WebSocketServer()
  server: Server;

  // handleConnection() is called when a client connects
  // OnGatewayConnection interface requires this method
  handleConnection(client: Socket) {
    console.log(`Client connected: ${client.id}`);

    // Emit a welcome message to the newly connected client
    client.emit("message", {
      text: "Welcome to the chat!",
      author: "System",
      timestamp: new Date(),
    });

    // Broadcast to all other clients that someone joined
    client.broadcast.emit("user-joined", {
      userId: client.id,
      timestamp: new Date(),
    });
  }

  // handleDisconnect() is called when a client disconnects
  handleDisconnect(client: Socket) {
    console.log(`Client disconnected: ${client.id}`);

    // Notify all other clients that someone left
    client.broadcast.emit("user-left", {
      userId: client.id,
      timestamp: new Date(),
    });
  }

  // @SubscribeMessage() handles messages sent from clients
  // 'message' is the event name clients will send
  @SubscribeMessage("message")
  handleMessage(
    @MessageBody() data: { text: string; author: string },
    @ConnectedSocket() client: Socket
  ) {
    console.log(`Message from ${client.id}:`, data);

    // Broadcast message to all connected clients (including sender)
    this.server.emit("message", {
      ...data,
      timestamp: new Date(),
    });

    // Return acknowledgment to sender
    return { status: "received" };
  }

  // Handle private messages (direct messaging)
  @SubscribeMessage("private-message")
  handlePrivateMessage(
    @MessageBody() data: { to: string; text: string; from: string },
    @ConnectedSocket() client: Socket
  ) {
    // Send message only to specific client
    this.server.to(data.to).emit("private-message", {
      from: data.from,
      text: data.text,
      timestamp: new Date(),
    });

    return { status: "sent" };
  }

  // Handle joining a room (for group chats)
  @SubscribeMessage("join-room")
  handleJoinRoom(
    @MessageBody() data: { room: string },
    @ConnectedSocket() client: Socket
  ) {
    client.join(data.room);

    // Notify others in the room
    client.to(data.room).emit("user-joined-room", {
      userId: client.id,
      room: data.room,
    });

    return { status: "joined", room: data.room };
  }

  // Broadcast to specific room
  @SubscribeMessage("room-message")
  handleRoomMessage(
    @MessageBody() data: { room: string; text: string; author: string },
    @ConnectedSocket() client: Socket
  ) {
    // Send message only to clients in the specified room
    this.server.to(data.room).emit("room-message", {
      ...data,
      timestamp: new Date(),
    });

    return { status: "sent" };
  }
}
```

### WebSocket Authentication

Authenticate WebSocket connections:

```typescript
// src/chat/chat.gateway.ts
import { UseGuards } from "@nestjs/common";
import { WsJwtGuard } from "../auth/guards/ws-jwt.guard";

@WebSocketGateway({ namespace: "/chat" })
export class ChatGateway {
  // Apply guard to all connections
  @UseGuards(WsJwtGuard)
  handleConnection(client: Socket) {
    // Client is authenticated if it reaches here
    const user = client.data.user; // Set by WsJwtGuard
    console.log(`Authenticated user connected: ${user.email}`);
  }
}
```

---

## Rate Limiting

Rate limiting prevents abuse by limiting the number of requests per time period.

### Install Rate Limiter

```bash
npm install @nestjs/throttler
```

### Configure Rate Limiter

```typescript
// src/app.module.ts
import { ThrottlerModule, ThrottlerGuard } from "@nestjs/throttler";
import { APP_GUARD } from "@nestjs/core";

@Module({
  imports: [
    // Configure rate limiting
    ThrottlerModule.forRoot({
      // ttl: Time window in seconds (60 = 1 minute)
      ttl: 60,

      // limit: Maximum number of requests per time window
      limit: 10, // 10 requests per minute
    }),
  ],
  providers: [
    // Apply rate limiting globally to all routes
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard,
    },
  ],
})
export class AppModule {}
```

### Custom Rate Limits

```typescript
// src/users/users.controller.ts
import { Throttle } from "@nestjs/throttler";

@Controller("users")
export class UsersController {
  // Override global rate limit for this endpoint
  @Throttle(5, 60) // 5 requests per 60 seconds
  @Post("register")
  register(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }
}
```

---

## API Versioning

API versioning allows you to maintain multiple API versions simultaneously.

### Enable Versioning

```typescript
// src/main.ts
import { VersioningType } from "@nestjs/common";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Enable versioning
  app.enableVersioning({
    type: VersioningType.URI, // Version in URL: /v1/users, /v2/users
    // Other options: VersioningType.HEADER, VersioningType.MEDIA_TYPE
  });

  await app.listen(3000);
}
bootstrap();
```

### Version Controllers

```typescript
// src/users/v1/users.controller.ts
@Controller({
  path: "users",
  version: "1",
})
export class UsersV1Controller {
  // Access at: GET /v1/users
  @Get()
  findAll() {
    return this.usersService.findAll();
  }
}

// src/users/v2/users.controller.ts
@Controller({
  path: "users",
  version: "2",
})
export class UsersV2Controller {
  // Access at: GET /v2/users
  @Get()
  findAll() {
    // New implementation with different response format
    return this.usersService.findAllV2();
  }
}
```

---

## CORS Configuration

CORS (Cross-Origin Resource Sharing) allows frontend apps on different domains to access your API.

```typescript
// src/main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Configure CORS
  app.enableCors({
    // origin: Allowed origins
    // In production, specify exact domains
    origin: process.env.FRONTEND_URL || "http://localhost:3001",

    // methods: Allowed HTTP methods
    methods: "GET,HEAD,PUT,PATCH,POST,DELETE,OPTIONS",

    // credentials: Allow cookies and auth headers
    credentials: true,

    // allowedHeaders: Allowed request headers
    allowedHeaders: "Content-Type, Authorization",
  });

  await app.listen(3000);
}
bootstrap();
```

---

## Best Practices

### 1. Use DTOs for All Inputs

```typescript
// Good: Type-safe and validated
@Post()
create(@Body() createUserDto: CreateUserDto) {}

// Bad: No validation or type safety
@Post()
create(@Body() user: any) {}
```

### 2. Return Appropriate Status Codes

```typescript
@Post()
@HttpCode(HttpStatus.CREATED) // 201
create() {}

@Delete(':id')
@HttpCode(HttpStatus.NO_CONTENT) // 204
remove() {}
```

### 3. Handle Errors Properly

```typescript
@Get(':id')
async findOne(@Param('id') id: string) {
  const user = await this.usersService.findOne(+id);
  if (!user) {
    throw new NotFoundException(`User with ID ${id} not found`);
  }
  return user;
}
```

### 4. Use Pagination for Lists

```typescript
@Get()
findAll(
  @Query('page') page: number = 1,
  @Query('limit') limit: number = 10,
) {
  return this.usersService.findAll({ page, limit });
}
```

---

## Mini Project: Real-time Chat Application

Build a real-time chat application using WebSockets.

### Features

1. WebSocket connection handling
2. Public chat room
3. Private messaging
4. User presence (online/offline)
5. Message history
6. Authentication for WebSocket connections

---

## Next Steps

✅ Understand RESTful API design  
✅ Document APIs with Swagger  
✅ Implement GraphQL queries  
✅ Build real-time features with WebSocket  
✅ Move to Module 8: File Handling & Media

---

_You now know how to build different types of APIs! 🚀_
