# Exception filters（NestJS 异常过滤器）

来源：[NestJS Exception filters 文档](https://docs.nestjs.com/exception-filters)

## Exception filters  异常过滤器

Nest 自带一个内置 **异常层**，负责处理应用中所有未处理的异常。当异常未被应用代码处理时，会被该层捕获，并自动发送合适的友好响应。

默认情况下，该行为由内置的 **全局异常过滤器** 执行，它会处理 `HttpException`（及其子类）异常。当异常 **无法识别**（既不是 `HttpException`，也不是继承自 `HttpException` 的类）时，内置异常过滤器会生成如下默认 JSON 响应：

```
{"statusCode":500,"message":"Internal server error"}
```

> **提示**
> 全局异常过滤器部分支持 `http-errors` 库。基本上，只要抛出的异常包含 `statusCode` 与 `message` 属性，就会被正确填充并作为响应返回（而不是对未知异常使用默认的 `InternalServerErrorException`）。

### Throwing standard exceptions  抛出标准异常

Nest 提供了内置的 `HttpException` 类，可从 `@nestjs/common` 包中使用。对典型的 HTTP REST/GraphQL API 应用，当出现某些错误条件时，最佳实践是返回标准 HTTP 响应对象。

例如，在 `CatsController` 中我们有一个 `findAll()` 方法（`GET` 路由处理器）。假设该处理器因为某种原因抛出异常，我们可以这样硬编码演示：

`cats.controller.ts`

```ts
@Get()
async findAll() {
  throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
}
```

> **提示**
> 我们在这里使用了 `HttpStatus`。这是从 `@nestjs/common` 包导入的辅助枚举。

当客户端调用该端点时，响应如下：

```
{"statusCode":403,"message":"Forbidden"}
```

`HttpException` 构造函数接收两个必需参数，用于决定响应：

- `response` 参数定义 JSON 响应体。可以是字符串或对象（如下所述）
- `status` 参数定义 [HTTP 状态码](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)

默认情况下，JSON 响应体包含两个属性：

- `statusCode`：默认为 `status` 参数提供的 HTTP 状态码
- `message`：基于 `status` 的简短错误描述

若只想覆盖 JSON 响应体中的 `message`，在 `response` 参数中传入字符串即可；若要覆盖整个 JSON 响应体，则传入对象，Nest 会序列化并返回该对象作为响应体。

第二个构造参数 `status` 应为有效的 HTTP 状态码，最佳实践是使用从 `@nestjs/common` 导入的 `HttpStatus` 枚举。

还有一个 **第三** 个可选构造参数 `options`，可提供错误的 [cause](https://nodejs.org/en/blog/release/v16.9.0/#error-cause)。该 `cause` 对象不会被序列化进响应体，但对日志记录很有用，可提供导致 `HttpException` 的内部错误信息。

下面示例覆盖整个响应体并提供错误原因：

`cats.controller.ts`

```ts
@Get()
async findAll() {
  try {
    await this.service.findAll();
  } catch (error) {
    throw new HttpException(
      {
        status: HttpStatus.FORBIDDEN,
        error: 'This is a custom message',
      },
      HttpStatus.FORBIDDEN,
      { cause: error },
    );
  }
}
```

使用以上方式，响应如下：

```
{"status":403,"error":"This is a custom message"}
```

### Exceptions logging  异常日志

默认情况下，异常过滤器不会记录 `HttpException`（及其子类）等内置异常。抛出这些异常时不会出现在控制台，因为它们被视为正常应用流程的一部分。其他内置异常，如 `WsException` 和 `RpcException`，同样如此。

这些异常都继承自 `IntrinsicException` 基类（从 `@nestjs/common` 导出）。该类用于区分“正常应用流程中的异常”和“非正常异常”。

若你希望记录这些异常，可以创建自定义异常过滤器。下一节会说明具体做法。

### Custom exceptions  自定义异常

很多情况下你并不需要自定义异常，可以直接使用内置的 Nest HTTP 异常（见下一节）。如果确实需要自定义异常，建议创建自己的 **异常层级**，让自定义异常继承 `HttpException` 基类。这样 Nest 就能识别你的异常，并自动处理错误响应。下面实现一个自定义异常：

`forbidden.exception.ts`

```ts
export class ForbiddenException extends HttpException {
  constructor() {
    super('Forbidden', HttpStatus.FORBIDDEN);
  }
}
```

由于 `ForbiddenException` 继承自 `HttpException`，它能与内置异常处理器无缝协作，我们因此可以在 `findAll()` 方法中使用它。

`cats.controller.ts`

```ts
@Get()
async findAll() {
  throw new ForbiddenException();
}
```

### Built-in HTTP exceptions  内置 HTTP 异常

Nest 提供了一组继承自 `HttpException` 基类的标准异常，这些从 `@nestjs/common` 包中导出，涵盖了常见 HTTP 异常：

- `BadRequestException`
- `UnauthorizedException`
- `NotFoundException`
- `ForbiddenException`
- `NotAcceptableException`
- `RequestTimeoutException`
- `ConflictException`
- `GoneException`
- `HttpVersionNotSupportedException`
- `PayloadTooLargeException`
- `UnsupportedMediaTypeException`
- `UnprocessableEntityException`
- `InternalServerErrorException`
- `NotImplementedException`
- `ImATeapotException`
- `MethodNotAllowedException`
- `BadGatewayException`
- `ServiceUnavailableException`
- `GatewayTimeoutException`
- `PreconditionFailedException`

所有内置异常还可以通过 `options` 参数提供错误 `cause` 和错误描述：

```
throw new BadRequestException('Something bad happened', {
  cause: new Error(),
  description: 'Some error description',
});
```

使用上面的代码，响应如下：

```
{"message":"Something bad happened","error":"Some error description","statusCode":400}
```

### Exception filters  异常过滤器

内置的异常过滤器可以自动处理许多场景，但你可能希望对异常层进行 **完全控制**。例如，你可能想添加日志，或根据动态因素采用不同的 JSON 结构。**异常过滤器** 正是为此设计的：它让你控制处理流程以及返回给客户端的响应内容。

我们创建一个异常过滤器，用于捕获 `HttpException` 实例，并为其实现自定义响应逻辑。为此需要访问底层平台的 `Request` 与 `Response` 对象。我们将从 `Request` 中取出原始 `url` 并记录到日志中，并使用 `Response` 的 `response.json()` 方法直接控制响应。

`http-exception.filter.ts`

```ts
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}
```

```js
import { Catch, HttpException } from '@nestjs/common';

@Catch(HttpException)
export class HttpExceptionFilter {
  catch(exception, host) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();
    const status = exception.getStatus();

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}
```

> **提示**
> 所有异常过滤器都应实现泛型 `ExceptionFilter<T>` 接口。这要求你提供具有指定签名的 `catch(exception: T, host: ArgumentsHost)` 方法。`T` 表示异常类型。

> **警告**
> 若使用 `@nestjs/platform-fastify`，可使用 `response.send()` 代替 `response.json()`。别忘了从 `fastify` 导入正确的类型。

`@Catch(HttpException)` 装饰器为异常过滤器绑定元数据，告诉 Nest 该过滤器只处理 `HttpException` 类型异常。`@Catch()` 可以接收单个参数或逗号分隔的列表，从而一次绑定多个异常类型。

### Arguments host  参数宿主

看看 `catch()` 方法参数：`exception` 是当前处理的异常对象，`host` 是 `ArgumentsHost` 对象。`ArgumentsHost` 是一个强大的工具对象，我们在[执行上下文](https://docs.nestjs.com/fundamentals/execution-context)章节会更深入讨论。在上述代码中，我们用它来获取传给原始请求处理器（控制器） 的 `Request` 与 `Response` 对象。我们使用 `ArgumentsHost` 的辅助方法获取所需对象。更多关于 `ArgumentsHost` 的内容请看[这里](https://docs.nestjs.com/fundamentals/execution-context)。

之所以需要这个抽象层，是因为 `ArgumentsHost` 可用于所有上下文（例如当前的 HTTP 服务器上下文，以及微服务和 WebSocket）。在执行上下文章节中我们会看到如何通过 `ArgumentsHost` 及其辅助函数访问 **任意** 执行上下文的[底层参数](https://docs.nestjs.com/fundamentals/execution-context#host-methods)。这使我们能编写跨上下文的通用异常过滤器。

## Learn the right way!  正确学习方式！

- 80+ 章节
- 5+ 小时视频
- 官方证书
- 深度研讨

[探索官方课程](https://courses.nestjs.com)

### Binding filters  绑定过滤器

让我们把新的 `HttpExceptionFilter` 绑定到 `CatsController` 的 `create()` 方法上。

`cats.controller.ts`

```ts
@Post()
@UseFilters(new HttpExceptionFilter())
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```

```js
@Post()
@UseFilters(new HttpExceptionFilter())
@Bind(Body())
async create(createCatDto) {
  throw new ForbiddenException();
}
```

> **提示**
> `@UseFilters()` 装饰器来自 `@nestjs/common` 包。

这里我们使用了 `@UseFilters()` 装饰器。类似 `@Catch()`，它可以接收单个过滤器实例或逗号分隔的多个实例。此处我们就地创建了 `HttpExceptionFilter` 实例。或者也可以传入类（而不是实例），由框架负责实例化，从而支持 **依赖注入**。

`cats.controller.ts`

```ts
@Post()
@UseFilters(HttpExceptionFilter)
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```

```js
@Post()
@UseFilters(HttpExceptionFilter)
@Bind(Body())
async create(createCatDto) {
  throw new ForbiddenException();
}
```

> **提示**
> 尽量使用类而不是实例应用过滤器。这会减少 **内存使用**，因为 Nest 可以在模块内复用类实例。

在上述例子中，`HttpExceptionFilter` 只应用于 `create()` 路由处理器，即方法级作用域。异常过滤器可用于不同作用域：方法级、控制器级或全局级。

例如，要设置控制器级过滤器：

`cats.controller.ts`

```ts
@Controller()
@UseFilters(new HttpExceptionFilter())
export class CatsController {}
```

这会让 `HttpExceptionFilter` 应用于 `CatsController` 中定义的每个路由处理器。

要创建全局作用域过滤器，可以这样做：

`main.ts`

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

> **警告**
> `useGlobalFilters()` 不会为网关或混合应用设置过滤器。

全局过滤器作用于整个应用，对所有控制器和路由处理器生效。在依赖注入方面，从任何模块之外注册的全局过滤器（如上使用 `useGlobalFilters()`）无法注入依赖，因为它发生在模块上下文之外。为了解决这个问题，可以 **直接从任何模块** 注册全局过滤器，使用如下构造：

`app.module.ts`

```ts
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```

> **提示**
> 使用此方式进行过滤器依赖注入时，请注意无论在哪个模块中使用该构造，过滤器实际上都是全局的。应放在过滤器（上例中的 `HttpExceptionFilter`）定义所在的模块中。另外，`useClass` 并非注册自定义提供者的唯一方式，更多请看[这里](https://docs.nestjs.com/fundamentals/custom-providers)。

你可以用此技术添加多个过滤器；只需将它们加入 providers 数组即可。

### Catch everything  捕获所有异常

要捕获 **所有** 未处理异常（无论类型），可将 `@Catch()` 的参数列表置空，例如 `@Catch()`。

下面示例使用 [HTTP 适配器](./faq/http-adapter) 来发送响应，因此它与平台无关，不直接使用平台特定的对象（`Request` 与 `Response`）：

```
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { HttpAdapterHost } from '@nestjs/core';

@Catch()
export class CatchEverythingFilter implements ExceptionFilter {
  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    // 在某些情况下，构造函数中可能无法获得 `httpAdapter`，因此在这里解析
    const { httpAdapter } = this.httpAdapterHost;
    const ctx = host.switchToHttp();
    const httpStatus =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const responseBody = {
      statusCode: httpStatus,
      timestamp: new Date().toISOString(),
      path: httpAdapter.getRequestUrl(ctx.getRequest()),
    };

    httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
  }
}
```

> **警告**
> 当把“捕获所有异常”的过滤器与绑定特定类型的过滤器组合使用时，应先声明“捕获所有异常”的过滤器，以便特定过滤器能正确处理其绑定类型。

### Inheritance  继承

通常你会创建完全定制的异常过滤器以满足应用需求。但在某些情况下，你可能只想扩展内置的默认 **全局异常过滤器**，并基于特定因素覆盖其行为。

要将异常处理委托给基础过滤器，需要继承 `BaseExceptionFilter` 并调用继承的 `catch()` 方法。

`all-exceptions.filter.ts`

```ts
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
```

```js
import { Catch } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception, host) {
    super.catch(exception, host);
  }
}
```

> **警告**
> 方法级和控制器级、且继承 `BaseExceptionFilter` 的过滤器不应使用 `new` 实例化，应由框架自动实例化。

全局过滤器 **可以** 扩展基础过滤器，可通过两种方式实现：

第一种方式是在实例化自定义全局过滤器时注入 `HttpAdapter` 引用：

```
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const { httpAdapter } = app.get(HttpAdapterHost);
  app.useGlobalFilters(new AllExceptionsFilter(httpAdapter));
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

第二种方式是使用 `APP_FILTER` token（见[绑定过滤器](exception-filters#binding-filters)部分）。
