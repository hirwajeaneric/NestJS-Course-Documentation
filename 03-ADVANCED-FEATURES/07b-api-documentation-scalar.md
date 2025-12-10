# Module 07b: API Documentation with Scalar

## 📚 Table of Contents
1. [Overview](#overview)
2. [What is Scalar?](#what-is-scalar)
3. [Installation & Setup](#installation--setup)
4. [Basic Configuration](#basic-configuration)
5. [Decorators & Annotations](#decorators--annotations)
6. [Advanced Features](#advanced-features)
7. [Customization](#customization)
8. [Authentication Documentation](#authentication-documentation)
9. [Examples & Schemas](#examples--schemas)
10. [Best Practices](#best-practices)
11. [Comparison with Swagger](#comparison-with-scalar)

---

## Overview

Scalar is a modern, beautiful, and developer-friendly API documentation tool that provides an alternative to Swagger UI. This module covers implementing comprehensive API documentation using Scalar in NestJS applications.

**Topics:**
- Scalar installation and setup
- OpenAPI/Swagger integration
- Decorators and annotations
- Customization options
- Authentication documentation
- Best practices

**Benefits:**
- Beautiful, modern UI
- Better developer experience
- Interactive API testing
- Automatic schema generation
- Type-safe documentation

---

## What is Scalar?

Scalar is an open-source API reference documentation tool that provides:
- **Modern UI**: Clean, intuitive interface
- **Interactive Testing**: Test APIs directly from documentation
- **Type Safety**: Better TypeScript support
- **Performance**: Fast and lightweight
- **Customization**: Highly customizable themes and layouts

**Key Features:**
- OpenAPI 3.0/3.1 support
- Interactive API explorer
- Code generation
- Dark mode support
- Mobile responsive

---

## Installation & Setup

### Install Dependencies

```bash
npm install @nestjs/swagger scalar-api-reference
```

### Basic Setup

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { ApiReference } from '@scalar/nestjs';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Swagger/OpenAPI Configuration
  const config = new DocumentBuilder()
    .setTitle('E-Commerce API')
    .setDescription('Complete API documentation for the E-Commerce platform')
    .setVersion('1.0')
    .addBearerAuth()
    .addTag('auth', 'Authentication endpoints')
    .addTag('users', 'User management')
    .addTag('products', 'Product management')
    .addTag('orders', 'Order management')
    .build();

  const document = SwaggerModule.createDocument(app, config);

  // Scalar API Reference
  app.use(
    '/api-docs',
    ApiReference({
      theme: 'default',
      layout: 'modern',
      spec: {
        content: document,
      },
    }),
  );

  await app.listen(3000);
}
bootstrap();
```

### Alternative: Using Scalar Standalone

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';
import { serve, setup } from 'swagger-ui-express';
import { apiReference } from '@scalar/api-reference';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('E-Commerce API')
    .setDescription('Complete API documentation')
    .setVersion('1.0')
    .addBearerAuth()
    .build();

  const document = SwaggerModule.createDocument(app, config);

  // Serve Scalar
  app.use(
    '/api-docs',
    apiReference({
      theme: 'default',
      spec: {
        content: document,
      },
    }),
  );

  await app.listen(3000);
}
bootstrap();
```

---

## Basic Configuration

### Document Builder Options

```typescript
const config = new DocumentBuilder()
  .setTitle('E-Commerce API')
  .setDescription('Complete API documentation for the E-Commerce platform')
  .setVersion('1.0')
  .setContact('Support', 'https://example.com', 'support@example.com')
  .setLicense('MIT', 'https://opensource.org/licenses/MIT')
  .addServer('https://api.example.com', 'Production')
  .addServer('https://staging-api.example.com', 'Staging')
  .addServer('http://localhost:3000', 'Local Development')
  .addBearerAuth(
    {
      type: 'http',
      scheme: 'bearer',
      bearerFormat: 'JWT',
      name: 'JWT',
      description: 'Enter JWT token',
      in: 'header',
    },
    'JWT-auth',
  )
  .addApiKey(
    {
      type: 'apiKey',
      name: 'X-API-Key',
      in: 'header',
      description: 'API Key for external services',
    },
    'api-key',
  )
  .addTag('auth', 'Authentication endpoints')
  .addTag('users', 'User management endpoints')
  .addTag('products', 'Product management endpoints')
  .addTag('orders', 'Order management endpoints')
  .build();
```

### Environment-Based Configuration

```typescript
// config/swagger.config.ts
import { DocumentBuilder } from '@nestjs/swagger';

export const swaggerConfig = (): DocumentBuilder => {
  const config = new DocumentBuilder()
    .setTitle('E-Commerce API')
    .setDescription('Complete API documentation')
    .setVersion('1.0')
    .addBearerAuth();

  if (process.env.NODE_ENV === 'production') {
    config.addServer('https://api.example.com', 'Production');
  } else if (process.env.NODE_ENV === 'staging') {
    config.addServer('https://staging-api.example.com', 'Staging');
  } else {
    config.addServer('http://localhost:3000', 'Local Development');
  }

  return config;
};
```

---

## Decorators & Annotations

### Controller Decorators

```typescript
// users.controller.ts
import { Controller, Get, Post, Body, Param } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse, ApiParam, ApiBody } from '@nestjs/swagger';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';
import { UserResponseDto } from './dto/user-response.dto';

@ApiTags('users')
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  @ApiOperation({ 
    summary: 'Create a new user',
    description: 'Creates a new user account with the provided information'
  })
  @ApiBody({ type: CreateUserDto })
  @ApiResponse({ 
    status: 201, 
    description: 'User successfully created',
    type: UserResponseDto 
  })
  @ApiResponse({ 
    status: 400, 
    description: 'Invalid input data' 
  })
  @ApiResponse({ 
    status: 409, 
    description: 'User already exists' 
  })
  async create(@Body() createUserDto: CreateUserDto): Promise<UserResponseDto> {
    return this.usersService.create(createUserDto);
  }

  @Get(':id')
  @ApiOperation({ summary: 'Get user by ID' })
  @ApiParam({ 
    name: 'id', 
    type: 'string', 
    description: 'User unique identifier',
    example: '123e4567-e89b-12d3-a456-426614174000'
  })
  @ApiResponse({ 
    status: 200, 
    description: 'User found',
    type: UserResponseDto 
  })
  @ApiResponse({ 
    status: 404, 
    description: 'User not found' 
  })
  async findOne(@Param('id') id: string): Promise<UserResponseDto> {
    return this.usersService.findOne(id);
  }
}
```

### DTO Decorators

```typescript
// dto/create-user.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { IsEmail, IsString, MinLength, MaxLength, IsOptional } from 'class-validator';

export class CreateUserDto {
  @ApiProperty({
    description: 'User email address',
    example: 'user@example.com',
    format: 'email',
  })
  @IsEmail()
  email: string;

  @ApiProperty({
    description: 'User password',
    example: 'SecurePassword123!',
    minLength: 8,
    maxLength: 50,
  })
  @IsString()
  @MinLength(8)
  @MaxLength(50)
  password: string;

  @ApiProperty({
    description: 'User full name',
    example: 'John Doe',
    required: false,
  })
  @IsString()
  @IsOptional()
  fullName?: string;

  @ApiProperty({
    description: 'User role',
    example: 'customer',
    enum: ['customer', 'admin', 'vendor'],
    default: 'customer',
  })
  @IsString()
  @IsOptional()
  role?: string;
}
```

### Response DTOs

```typescript
// dto/user-response.dto.ts
import { ApiProperty } from '@nestjs/swagger';

export class UserResponseDto {
  @ApiProperty({
    description: 'User unique identifier',
    example: '123e4567-e89b-12d3-a456-426614174000',
  })
  id: string;

  @ApiProperty({
    description: 'User email address',
    example: 'user@example.com',
  })
  email: string;

  @ApiProperty({
    description: 'User full name',
    example: 'John Doe',
    required: false,
  })
  fullName?: string;

  @ApiProperty({
    description: 'User role',
    example: 'customer',
    enum: ['customer', 'admin', 'vendor'],
  })
  role: string;

  @ApiProperty({
    description: 'Account creation date',
    example: '2024-01-15T10:30:00Z',
  })
  createdAt: Date;

  @ApiProperty({
    description: 'Last update date',
    example: '2024-01-20T14:45:00Z',
  })
  updatedAt: Date;
}
```

### Pagination DTOs

```typescript
// dto/pagination.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { Type } from 'class-transformer';
import { IsOptional, IsInt, Min, Max } from 'class-validator';

export class PaginationDto {
  @ApiProperty({
    description: 'Page number (1-indexed)',
    example: 1,
    minimum: 1,
    default: 1,
    required: false,
  })
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page?: number = 1;

  @ApiProperty({
    description: 'Number of items per page',
    example: 10,
    minimum: 1,
    maximum: 100,
    default: 10,
    required: false,
  })
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  limit?: number = 10;
}

export class PaginatedResponseDto<T> {
  @ApiProperty({ description: 'Array of items' })
  data: T[];

  @ApiProperty({ description: 'Total number of items', example: 100 })
  total: number;

  @ApiProperty({ description: 'Current page number', example: 1 })
  page: number;

  @ApiProperty({ description: 'Number of items per page', example: 10 })
  limit: number;

  @ApiProperty({ description: 'Total number of pages', example: 10 })
  totalPages: number;
}
```

---

## Advanced Features

### File Upload Documentation

```typescript
// products.controller.ts
import { ApiTags, ApiOperation, ApiConsumes, ApiBody } from '@nestjs/swagger';
import { FileInterceptor } from '@nestjs/platform-express';

@ApiTags('products')
@Controller('products')
export class ProductsController {
  @Post(':id/image')
  @UseInterceptors(FileInterceptor('file'))
  @ApiOperation({ summary: 'Upload product image' })
  @ApiConsumes('multipart/form-data')
  @ApiBody({
    schema: {
      type: 'object',
      properties: {
        file: {
          type: 'string',
          format: 'binary',
          description: 'Product image file (JPEG, PNG, max 5MB)',
        },
      },
    },
  })
  async uploadImage(
    @Param('id') id: string,
    @UploadedFile() file: Express.Multer.File,
  ) {
    return this.productsService.uploadImage(id, file);
  }
}
```

### Query Parameters

```typescript
// products.controller.ts
import { ApiQuery } from '@nestjs/swagger';

@Get()
@ApiOperation({ summary: 'Get all products with filters' })
@ApiQuery({
  name: 'category',
  required: false,
  type: String,
  description: 'Filter by category',
  example: 'electronics',
})
@ApiQuery({
  name: 'minPrice',
  required: false,
  type: Number,
  description: 'Minimum price filter',
  example: 10,
})
@ApiQuery({
  name: 'maxPrice',
  required: false,
  type: Number,
  description: 'Maximum price filter',
  example: 1000,
})
@ApiQuery({
  name: 'sortBy',
  required: false,
  enum: ['price', 'name', 'createdAt'],
  description: 'Sort field',
  example: 'price',
})
@ApiQuery({
  name: 'order',
  required: false,
  enum: ['ASC', 'DESC'],
  description: 'Sort order',
  example: 'ASC',
})
async findAll(@Query() query: ProductQueryDto) {
  return this.productsService.findAll(query);
}
```

### Array Responses

```typescript
@Get()
@ApiOperation({ summary: 'Get all users' })
@ApiResponse({
  status: 200,
  description: 'List of users',
  type: [UserResponseDto],
})
async findAll(): Promise<UserResponseDto[]> {
  return this.usersService.findAll();
}
```

### Nested Objects

```typescript
// dto/create-order.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { Type } from 'class-transformer';
import { IsArray, ValidateNested } from 'class-validator';

export class OrderItemDto {
  @ApiProperty({ description: 'Product ID', example: 'prod-123' })
  productId: string;

  @ApiProperty({ description: 'Quantity', example: 2, minimum: 1 })
  quantity: number;

  @ApiProperty({ description: 'Price per item', example: 29.99 })
  price: number;
}

export class CreateOrderDto {
  @ApiProperty({ description: 'Shipping address' })
  shippingAddress: string;

  @ApiProperty({
    description: 'Order items',
    type: [OrderItemDto],
    example: [
      { productId: 'prod-123', quantity: 2, price: 29.99 },
      { productId: 'prod-456', quantity: 1, price: 49.99 },
    ],
  })
  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => OrderItemDto)
  items: OrderItemDto[];
}
```

---

## Customization

### Custom Theme Configuration

```typescript
// main.ts
app.use(
  '/api-docs',
  ApiReference({
    theme: 'purple',
    layout: 'modern',
    spec: {
      content: document,
    },
    configuration: {
      theme: 'purple',
      layout: 'modern',
      hideModels: false,
      hideDownloadButton: false,
      hideTryIt: false,
      hideSchema: false,
      hideSidebar: false,
      hideSearch: false,
      hideInfo: false,
      hideServers: false,
      hideAuthentication: false,
      defaultHttpClient: {
        targetKey: 'javascript',
        clientKey: 'fetch',
      },
    },
  }),
);
```

### Custom CSS Styling

```typescript
app.use(
  '/api-docs',
  ApiReference({
    theme: 'default',
    layout: 'modern',
    spec: {
      content: document,
    },
    customCss: `
      .scalar-api-reference {
        --scalar-color-1: #6366f1;
        --scalar-color-2: #8b5cf6;
        --scalar-color-3: #ec4899;
      }
    `,
  }),
);
```

### Multiple API Versions

```typescript
// main.ts
const v1Document = SwaggerModule.createDocument(app, v1Config, {
  include: [V1Module],
});

const v2Document = SwaggerModule.createDocument(app, v2Config, {
  include: [V2Module],
});

app.use('/api-docs/v1', ApiReference({ spec: { content: v1Document } }));
app.use('/api-docs/v2', ApiReference({ spec: { content: v2Document } }));
```

---

## Authentication Documentation

### JWT Bearer Authentication

```typescript
// main.ts
const config = new DocumentBuilder()
  .addBearerAuth(
    {
      type: 'http',
      scheme: 'bearer',
      bearerFormat: 'JWT',
      name: 'JWT',
      description: 'Enter JWT token',
      in: 'header',
    },
    'JWT-auth',
  )
  .build();
```

### Using Authentication in Controllers

```typescript
// users.controller.ts
import { ApiBearerAuth, ApiSecurity } from '@nestjs/swagger';

@ApiTags('users')
@ApiBearerAuth('JWT-auth')
@Controller('users')
export class UsersController {
  @Get('profile')
  @ApiOperation({ summary: 'Get current user profile' })
  @ApiResponse({ status: 200, type: UserResponseDto })
  @ApiResponse({ status: 401, description: 'Unauthorized' })
  async getProfile(@CurrentUser() user: User) {
    return user;
  }
}
```

### API Key Authentication

```typescript
// main.ts
const config = new DocumentBuilder()
  .addApiKey(
    {
      type: 'apiKey',
      name: 'X-API-Key',
      in: 'header',
      description: 'API Key for external services',
    },
    'api-key',
  )
  .build();
```

```typescript
// external.controller.ts
@ApiSecurity('api-key')
@Controller('external')
export class ExternalController {
  // Protected endpoints
}
```

### OAuth2 Documentation

```typescript
// main.ts
const config = new DocumentBuilder()
  .addOAuth2(
    {
      type: 'oauth2',
      flows: {
        authorizationCode: {
          authorizationUrl: 'https://example.com/oauth/authorize',
          tokenUrl: 'https://example.com/oauth/token',
          scopes: {
            read: 'Read access',
            write: 'Write access',
          },
        },
      },
    },
    'oauth2',
  )
  .build();
```

---

## Examples & Schemas

### Providing Examples

```typescript
// dto/create-product.dto.ts
import { ApiProperty } from '@nestjs/swagger';

export class CreateProductDto {
  @ApiProperty({
    description: 'Product name',
    example: 'iPhone 15 Pro',
    minLength: 3,
    maxLength: 100,
  })
  name: string;

  @ApiProperty({
    description: 'Product description',
    example: 'Latest iPhone with advanced features',
    required: false,
  })
  description?: string;

  @ApiProperty({
    description: 'Product price in USD',
    example: 999.99,
    minimum: 0,
    type: Number,
  })
  price: number;

  @ApiProperty({
    description: 'Product category',
    example: 'electronics',
    enum: ['electronics', 'clothing', 'books', 'home'],
  })
  category: string;

  @ApiProperty({
    description: 'Stock quantity',
    example: 50,
    minimum: 0,
    type: Number,
  })
  stock: number;
}
```

### Complex Examples

```typescript
@ApiProperty({
  description: 'Product specifications',
  example: {
    color: 'Space Gray',
    storage: '256GB',
    screenSize: '6.1 inches',
    weight: '187g',
  },
  type: 'object',
  additionalProperties: true,
})
specifications: Record<string, any>;
```

### Enum Documentation

```typescript
export enum OrderStatus {
  PENDING = 'pending',
  PROCESSING = 'processing',
  SHIPPED = 'shipped',
  DELIVERED = 'delivered',
  CANCELLED = 'cancelled',
}

export class OrderResponseDto {
  @ApiProperty({
    description: 'Order status',
    enum: OrderStatus,
    example: OrderStatus.PENDING,
  })
  status: OrderStatus;
}
```

---

## Best Practices

### 1. Comprehensive Documentation

```typescript
// ✅ Good: Complete documentation
@Post()
@ApiOperation({ 
  summary: 'Create a new user',
  description: 'Creates a new user account with email and password. Returns the created user object with generated ID.'
})
@ApiBody({ type: CreateUserDto })
@ApiResponse({ 
  status: 201, 
  description: 'User successfully created',
  type: UserResponseDto 
})
@ApiResponse({ 
  status: 400, 
  description: 'Invalid input data. Check validation errors.' 
})
@ApiResponse({ 
  status: 409, 
  description: 'User with this email already exists' 
})
async create(@Body() createUserDto: CreateUserDto) {
  // ...
}

// ❌ Bad: Missing documentation
@Post()
async create(@Body() createUserDto: CreateUserDto) {
  // ...
}
```

### 2. Consistent DTOs

```typescript
// ✅ Good: Separate DTOs for request and response
export class CreateUserDto { }
export class UserResponseDto { }
export class UpdateUserDto { }

// ❌ Bad: Reusing entity classes directly
export class User { }
```

### 3. Error Response Documentation

```typescript
// dto/error-response.dto.ts
import { ApiProperty } from '@nestjs/swagger';

export class ErrorResponseDto {
  @ApiProperty({
    description: 'Error status code',
    example: 400,
  })
  statusCode: number;

  @ApiProperty({
    description: 'Error message',
    example: 'Validation failed',
  })
  message: string | string[];

  @ApiProperty({
    description: 'Error type',
    example: 'Bad Request',
  })
  error: string;

  @ApiProperty({
    description: 'Timestamp of the error',
    example: '2024-01-15T10:30:00Z',
  })
  timestamp: string;

  @ApiProperty({
    description: 'Request path',
    example: '/api/users',
  })
  path: string;
}
```

### 4. Versioning Documentation

```typescript
// v1/users.controller.ts
@ApiTags('users-v1')
@Controller('v1/users')
export class UsersV1Controller { }

// v2/users.controller.ts
@ApiTags('users-v2')
@Controller('v2/users')
export class UsersV2Controller { }
```

### 5. Group Related Endpoints

```typescript
@ApiTags('authentication')
@Controller('auth')
export class AuthController {
  @Post('register')
  @ApiOperation({ summary: 'Register new user' })
  // ...

  @Post('login')
  @ApiOperation({ summary: 'Login user' })
  // ...

  @Post('refresh')
  @ApiOperation({ summary: 'Refresh access token' })
  // ...
}
```

### 6. Document Pagination

```typescript
@Get()
@ApiOperation({ summary: 'Get paginated list of products' })
@ApiQuery({ name: 'page', required: false, type: Number })
@ApiQuery({ name: 'limit', required: false, type: Number })
@ApiResponse({
  status: 200,
  description: 'Paginated list of products',
  type: PaginatedResponseDto,
})
async findAll(@Query() pagination: PaginationDto) {
  // ...
}
```

### 7. Security Documentation

```typescript
@ApiBearerAuth('JWT-auth')
@ApiOperation({ summary: 'Get user profile (requires authentication)' })
@ApiResponse({ status: 200, type: UserResponseDto })
@ApiResponse({ status: 401, description: 'Unauthorized - Invalid or missing token' })
async getProfile() {
  // ...
}
```

---

## Comparison with Swagger

### Scalar vs Swagger UI

| Feature | Scalar | Swagger UI |
|---------|--------|------------|
| UI Design | Modern, clean | Traditional |
| Performance | Fast | Slower |
| TypeScript Support | Excellent | Good |
| Customization | High | Medium |
| Interactive Testing | Yes | Yes |
| Dark Mode | Built-in | Limited |
| Mobile Support | Excellent | Good |
| Code Generation | Yes | Yes |
| OpenAPI Support | 3.0/3.1 | 3.0/3.1 |

### When to Use Scalar

- ✅ Modern, developer-friendly UI is important
- ✅ Better TypeScript integration needed
- ✅ Want better performance
- ✅ Need extensive customization
- ✅ Mobile-friendly documentation is required

### When to Use Swagger UI

- ✅ Team is already familiar with Swagger
- ✅ Need extensive plugin ecosystem
- ✅ Legacy system integration
- ✅ Specific Swagger features required

---

## Mini Project: Document a Complete API

### Project Requirements

Create comprehensive API documentation for a Blog API with the following endpoints:

1. **Authentication**
   - POST `/auth/register` - Register new user
   - POST `/auth/login` - Login user
   - POST `/auth/refresh` - Refresh token

2. **Posts**
   - GET `/posts` - Get all posts (paginated)
   - GET `/posts/:id` - Get post by ID
   - POST `/posts` - Create post (authenticated)
   - PUT `/posts/:id` - Update post (authenticated)
   - DELETE `/posts/:id` - Delete post (authenticated)

3. **Comments**
   - GET `/posts/:id/comments` - Get comments for a post
   - POST `/posts/:id/comments` - Add comment (authenticated)

### Implementation Checklist

- [ ] Install Scalar and Swagger dependencies
- [ ] Configure Swagger document builder
- [ ] Set up Scalar API reference
- [ ] Create DTOs with proper decorators
- [ ] Document all endpoints with examples
- [ ] Add authentication documentation
- [ ] Document error responses
- [ ] Add pagination documentation
- [ ] Customize Scalar theme
- [ ] Test interactive API explorer

### Expected Outcome

A fully documented API with:
- Complete endpoint documentation
- Request/response examples
- Authentication flow
- Error handling documentation
- Interactive testing capability
- Beautiful, modern UI

---

## Summary

In this module, you learned:

✅ How to install and configure Scalar API documentation  
✅ Using Swagger decorators for comprehensive documentation  
✅ Documenting DTOs, controllers, and responses  
✅ Authentication and security documentation  
✅ Advanced features like file uploads and pagination  
✅ Customization options for Scalar  
✅ Best practices for API documentation  

**Next Steps:**
- Document your main project APIs
- Customize Scalar theme to match your brand
- Add comprehensive examples for all endpoints
- Set up automated documentation generation in CI/CD

---

*Continue to the next module to learn about production deployment strategies!*

