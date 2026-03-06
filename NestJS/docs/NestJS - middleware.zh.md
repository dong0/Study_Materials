# Middleware（NestJS 中间件）

来源：[NestJS Middleware 文档](https://docs.nestjs.com/middleware)

## Middleware  中间件

中间件是在路由处理器 **之前** 调用的函数。中间件函数可以访问 [request](https://expressjs.com/en/4x/api.html#req) 与 [response](https://expressjs.com/en/4x/api.html#res) 对象，以及应用请求-响应周期中的 `next()` 中间件函数。**next** 中间件函数通常用名为 `next` 的变量表示。

Nest 中间件默认等同于 [Express](https://expressjs.com/en/guide/using-middleware.html) 中间件。下面摘自官方 Express 文档，对中间件能力的描述：

> 中间件函数可以执行以下任务：  
> - 执行任何代码  
> - 修改请求与响应对象  
> - 结束请求-响应周期  
> - 调用堆栈中的下一个中间件函数  
> - 如果当前中间件未结束请求-响应周期，就必须调用 `next()` 将控制权交给下一个中间件，否则请求会被悬挂

你可以用函数或带 `@Injectable()` 装饰器的类实现自定义 Nest 中间件。类应实现 `NestMiddleware` 接口，而函数没有特殊要求。先用类的方式实现一个简单的中间件。

> **警告**
> `Express` 与 `fastify` 对中间件的处理方式不同，方法签名也不同，详见[这里](https://docs.nestjs.com/techniques/performance#middleware)。

`logger.middleware.ts`

```ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log('Request...');
    next();
  }
}
```

```js
import { Injectable } from '@nestjs/common';

@Injectable()
export class LoggerMiddleware {
  use(req, res, next) {
    console.log('Request...');
    next();
  }
}
```

### Dependency injection  依赖注入

Nest 中间件完全支持依赖注入。与提供者和控制器一样，它们可以 **注入** 同一模块中的依赖，通常通过 `constructor` 完成。

### Applying middleware  应用中间件

`@Module()` 装饰器中没有中间件的位置。相反，我们通过模块类的 `configure()` 方法进行设置。包含中间件的模块必须实现 `NestModule` 接口。下面在 `AppModule` 级别设置 `LoggerMiddleware`。

`app.module.ts`

```ts
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes('cats');
  }
}
```

```js
import { Module } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {
  configure(consumer) {
    consumer.apply(LoggerMiddleware).forRoutes('cats');
  }
}
```

上例中，我们把 `LoggerMiddleware` 绑定到之前在 `CatsController` 中定义的 `/cats` 路由处理器上。也可以通过向 `forRoutes()` 传入包含路由 `path` 与请求 `method` 的对象，进一步将中间件限制到某个请求方法。下例中引入了 `RequestMethod` 枚举以引用请求方法类型：

`app.module.ts`

```ts
import { Module, NestModule, RequestMethod, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes({ path: 'cats', method: RequestMethod.GET });
  }
}
```

```js
import { Module, RequestMethod } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {
  configure(consumer) {
    consumer.apply(LoggerMiddleware).forRoutes({ path: 'cats', method: RequestMethod.GET });
  }
}
```

> **提示**
> `configure()` 方法可以使用 `async/await` 做成异步（例如在方法体中 `await` 某个异步操作完成）。

> **警告**
> 使用 `express` 适配器时，NestJS 默认会通过 `body-parser` 注册 `json` 和 `urlencoded` 中间件。如果你想通过 `MiddlewareConsumer` 自定义该中间件，需要在 `NestFactory.create()` 创建应用时将 `bodyParser` 设为 `false` 以关闭全局中间件。

### Route wildcards  路由通配符

NestJS 中间件也支持基于模式的路由。例如，可使用命名通配符（`\*splat`）匹配任意字符组合。下面示例中，中间件会对所有以 `abcd/` 开头的路由执行，无论后续有多少字符。

```
forRoutes({ path: 'abcd/*splat', method: RequestMethod.ALL });
```

> **提示**
> `splat` 只是通配符参数名，没有特殊含义。你可以命名为任意名称，例如 `*wildcard`。

`'abcd/*'` 路由路径会匹配 `abcd/1`、`abcd/123`、`abcd/abc` 等。连字符（`-`）与点号（`.`）在字符串路径中按字面解析。然而 `abcd/`（无额外字符）不会匹配。若要匹配它，需要用花括号包裹通配符使其可选：

```
forRoutes({ path: 'abcd/{*splat}', method: RequestMethod.ALL });
```

### Middleware consumer  中间件消费者

`MiddlewareConsumer` 是一个辅助类，提供多个内置方法来管理中间件。它们都可以采用 [流式风格](https://en.wikipedia.org/wiki/Fluent_interface) **链式** 调用。`forRoutes()` 可接收单个字符串、多个字符串、`RouteInfo` 对象、控制器类或多个控制器类。多数情况下，你可能会传入以逗号分隔的 **控制器** 列表。下面示例展示了使用单个控制器：

`app.module.ts`

```ts
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes(CatsController);
  }
}
```

```js
import { Module } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

@Module({
  imports: [CatsModule],
})
export class AppModule {
  configure(consumer) {
    consumer.apply(LoggerMiddleware).forRoutes(CatsController);
  }
}
```

> **提示**
> `apply()` 方法既可以接收单个中间件，也可以接收多个参数来指定[多个中间件](https://docs.nestjs.com/middleware#multiple-middleware)。

### Excluding routes  排除路由

有时我们希望 **排除** 某些路由不应用中间件。这可以通过 `exclude()` 方法轻松实现。`exclude()` 接受单个字符串、多个字符串或 `RouteInfo` 对象，用于标识要排除的路由。

示例如下：

```
consumer
  .apply(LoggerMiddleware)
  .exclude(
    { path: 'cats', method: RequestMethod.GET },
    { path: 'cats', method: RequestMethod.POST },
    'cats/*splat',
  )
  .forRoutes(CatsController);
```

> **提示**
> `exclude()` 支持使用 [path-to-regexp](https://github.com/pillarjs/path-to-regexp#parameters) 包的通配符参数。

以上例子中，`LoggerMiddleware` 会绑定到 `CatsController` 中定义的所有路由，**除了** `exclude()` 中传入的三条路由。

该方法提供了基于特定路由或路由模式灵活应用或排除中间件的能力。

### Functional middleware  函数式中间件

我们一直在使用的 `LoggerMiddleware` 类非常简单：没有成员变量、没有额外方法、没有依赖。那为什么不直接用函数定义它呢？确实可以。这类中间件称为 **函数式中间件**。我们将类中间件改造成函数式中间件来展示差异：

`logger.middleware.ts`

```ts
import { Request, Response, NextFunction } from 'express';

export function logger(req: Request, res: Response, next: NextFunction) {
  console.log('Request...');
  next();
}
```

```js
export function logger(req, res, next) {
  console.log('Request...');
  next();
}
```

并在 `AppModule` 中使用：

`app.module.ts`

```ts
consumer.apply(logger).forRoutes(CatsController);
```

> **提示**
> 当中间件不需要任何依赖时，考虑使用更简单的 **函数式中间件**。

### Multiple middleware  多个中间件

如前所述，要绑定多个按顺序执行的中间件，只需在 `apply()` 方法中传入以逗号分隔的列表：

```
consumer.apply(cors(), helmet(), logger).forRoutes(CatsController);
```

### Global middleware  全局中间件

若要一次性将中间件绑定到所有已注册路由，可使用 `INestApplication` 实例提供的 `use()` 方法：

`main.ts`

```ts
const app = await NestFactory.create(AppModule);
app.use(logger);
await app.listen(process.env.PORT ?? 3000);
```

> **提示**
> 在全局中间件中无法访问 DI 容器。使用 `app.use()` 时可以改用[函数式中间件](middleware#functional-middleware)。或者，你也可以使用类中间件，并在 `AppModule`（或其他模块）中通过 `.forRoutes('*')` 来消费。
