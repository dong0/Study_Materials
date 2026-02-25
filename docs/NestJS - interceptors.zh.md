# Interceptors（NestJS 拦截器）

来源：[NestJS Interceptors 文档](https://docs.nestjs.com/interceptors)

## Interceptors  拦截器

拦截器是一个使用 `@Injectable()` 装饰器标注并实现 `NestInterceptor` 接口的类。

拦截器具有一组有用的能力，这些能力受到 [面向切面编程](https://en.wikipedia.org/wiki/Aspect-oriented_programming)（AOP）技术的启发。它们可以：

- 在方法执行前 / 后绑定额外逻辑
- 转换函数返回结果
- 转换函数抛出的异常
- 扩展基本函数行为
- 在特定条件下完全覆盖函数（例如用于缓存）

## Basics  基础

每个拦截器都实现 `intercept()` 方法，该方法接收两个参数。第一个参数是 `ExecutionContext` 实例（与 [守卫](https://docs.nestjs.com/guards) 使用的对象完全相同）。`ExecutionContext` 继承自 `ArgumentsHost`。我们在异常过滤器章节中见过 `ArgumentsHost`。在那里我们看到它是对传递给原始处理器的参数的封装，并且会根据应用类型包含不同的参数数组。可参考 [异常过滤器](https://docs.nestjs.com/exception-filters#arguments-host) 了解更多。

## Execution context  执行上下文

通过扩展 `ArgumentsHost`，`ExecutionContext` 还添加了若干新的辅助方法，用于提供当前执行过程的更多细节。这些细节有助于构建更通用的拦截器，使其可跨一组更广泛的控制器、方法和执行上下文工作。了解更多 `ExecutionContext` 的内容请看 [这里](https://docs.nestjs.com/fundamentals/execution-context)。

## Call handler  调用处理器

第二个参数是 `CallHandler`。`CallHandler` 接口实现了 `handle()` 方法，你可以在拦截器中的某个时点调用它来执行路由处理方法。如果你在 `intercept()` 实现中不调用 `handle()`，路由处理方法将不会被执行。

这种方式意味着 `intercept()` 方法实际上 **包裹** 了请求/响应流。因此，你可以在最终路由处理器执行 **之前和之后** 实现自定义逻辑。很明显，你可以在 `intercept()` 中写在调用 `handle()` **之前** 执行的代码，但如何影响 **之后** 的行为呢？因为 `handle()` 返回一个 `Observable`，我们可以使用强大的 [RxJS](https://github.com/ReactiveX/rxjs) 操作符进一步处理响应。用 AOP 术语来说，调用路由处理器（即调用 `handle()`）称为 [Pointcut](https://en.wikipedia.org/wiki/Pointcut)，它表示我们的额外逻辑插入的点。

例如，考虑一个传入的 `POST /cats` 请求。该请求会到达 `CatsController` 中定义的 `create()` 处理器。如果在调用链中某个拦截器没有调用 `handle()`，`create()` 方法将不会执行。一旦调用 `handle()`（并返回其 `Observable`），`create()` 处理器就会被触发。而当响应流通过 `Observable` 返回时，可以对该流执行额外操作，并向调用方返回最终结果。

## Explore your graph with NestJS Devtools  使用 NestJS Devtools 探索你的图谱

- 图谱可视化
- 路由导航
- 交互式演练场
- CI/CD 集成

[注册](https://devtools.nestjs.com)

## Aspect interception  切面拦截

我们要看的第一个用例是使用拦截器记录用户交互（例如，存储用户调用、异步派发事件或计算时间戳）。下面展示一个简单的 `LoggingInterceptor`：

`logging.interceptor.ts`

```ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');
    const now = Date.now();
    return next.handle().pipe(
      tap(() => console.log(`After... ${Date.now() - now}ms`)),
    );
  }
}
```

```js
import { Injectable } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor {
  intercept(context, next) {
    console.log('Before...');
    const now = Date.now();
    return next.handle().pipe(
      tap(() => console.log(`After... ${Date.now() - now}ms`)),
    );
  }
}
```

> **提示**
> `NestInterceptor<T, R>` 是一个泛型接口，其中 `T` 表示 `Observable<T>` 的类型（支持响应流），而 `R` 是 `Observable<R>` 包装的值的类型。

> **注意**
> 拦截器和控制器、提供者、守卫等一样，可以通过 `constructor` 注入依赖。

由于 `handle()` 返回 RxJS 的 `Observable`，我们可以使用多种操作符来操作该流。在上面的例子中，我们使用了 `tap()` 操作符，它会在可观察流正常结束或异常结束时调用我们的匿名日志函数，但不会影响响应流程的其他部分。

## Binding interceptors  绑定拦截器

要设置拦截器，我们使用从 `@nestjs/common` 包导入的 `@UseInterceptors()` 装饰器。和 [管道](https://docs.nestjs.com/pipes)、[守卫](https://docs.nestjs.com/guards) 一样，拦截器可以是控制器级、方法级或全局级。

`cats.controller.ts`

```ts
@UseInterceptors(LoggingInterceptor)
export class CatsController {}
```

> **提示**
> `@UseInterceptors()` 装饰器从 `@nestjs/common` 包导入。

通过上述构造，`CatsController` 中定义的每个路由处理器都会使用 `LoggingInterceptor`。当有人调用 `GET /cats` 端点时，你会在标准输出中看到如下内容：

```
Before...
After... 1ms
```

注意这里我们传入的是 `LoggingInterceptor` 类（而不是实例），由框架负责实例化，从而支持依赖注入。与管道、守卫和异常过滤器一样，我们也可以传入就地实例：

`cats.controller.ts`

```ts
@UseInterceptors(new LoggingInterceptor())
export class CatsController {}
```

如前所述，上面的构造会将拦截器绑定到该控制器声明的每个处理器上。如果我们想将拦截器的作用域限制到单个方法，只需在 **方法级** 使用该装饰器。

要设置全局拦截器，可以使用 Nest 应用实例的 `useGlobalInterceptors()` 方法：

```ts
const app = await NestFactory.create(AppModule);
app.useGlobalInterceptors(new LoggingInterceptor());
```

全局拦截器作用于整个应用中的每个控制器和每个路由处理器。在依赖注入方面，从任何模块之外注册的全局拦截器（如上述使用 `useGlobalInterceptors()` 的方式）无法注入依赖，因为它发生在任何模块上下文之外。为了解决这个问题，你可以 **直接从任何模块** 设置拦截器，使用如下构造：

`app.module.ts`

```ts
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```

> **提示**
> 当使用这种方式为拦截器进行依赖注入时，请注意无论该构造放在哪个模块中，拦截器实际上都是全局的。应该放在哪里？选择拦截器（上例中的 `LoggingInterceptor`）定义所在的模块。另外，`useClass` 不是注册自定义提供者的唯一方式。了解更多请看 [这里](https://docs.nestjs.com/fundamentals/custom-providers)。

## Response mapping  响应映射

我们已经知道 `handle()` 会返回 `Observable`。该流包含了路由处理器 **返回** 的值，因此我们可以很容易使用 RxJS 的 `map()` 操作符进行变换。

> **警告**
> 响应映射功能不适用于库特定的响应策略（直接使用 `@Res()` 对象是被禁止的）。

让我们创建 `TransformInterceptor`，用一个简单的方式修改每个响应来演示这个过程。它将使用 RxJS 的 `map()` 操作符把响应对象赋值给新对象的 `data` 属性，并将该新对象返回给客户端。

`transform.interceptor.ts`

```ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  data: T;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(map(data => ({ data })));
  }
}
```

```js
import { Injectable } from '@nestjs/common';
import { map } from 'rxjs/operators';

@Injectable()
export class TransformInterceptor {
  intercept(context, next) {
    return next.handle().pipe(map(data => ({ data })));
  }
}
```

> **提示**
> Nest 拦截器既支持同步的 `intercept()` 方法，也支持异步的 `intercept()` 方法。如果需要，你可以将该方法改为 `async`。

通过上述构造，当有人调用 `GET /cats` 端点时，响应会如下所示（假设路由处理器返回空数组 `[]`）：

```
{"data":[]}
```

拦截器在创建跨整个应用的可复用解决方案方面非常有价值。

例如，假设我们需要将每个 `null` 值转换成空字符串 `''`。我们可以用一行代码实现，并将该拦截器全局绑定，使其自动应用于每个已注册的处理器。

`exclude-null.interceptor.ts`

```ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class ExcludeNullInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(map(value => value === null ? '' : value));
  }
}
```

```js
import { Injectable } from '@nestjs/common';
import { map } from 'rxjs/operators';

@Injectable()
export class ExcludeNullInterceptor {
  intercept(context, next) {
    return next.handle().pipe(map(value => value === null ? '' : value));
  }
}
```

## Exception mapping  异常映射

另一个有意思的用例是使用 RxJS 的 `catchError()` 操作符来覆盖抛出的异常：

`errors.interceptor.ts`

```ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  BadGatewayException,
  CallHandler,
} from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      catchError(err => throwError(() => new BadGatewayException())),
    );
  }
}
```

```js
import { Injectable, BadGatewayException } from '@nestjs/common';
import { throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor {
  intercept(context, next) {
    return next.handle().pipe(
      catchError(err => throwError(() => new BadGatewayException())),
    );
  }
}
```

## Stream overriding  流覆盖

有几个原因会让我们想完全阻止调用处理器并返回不同的值。一个显而易见的例子是实现缓存以提升响应时间。来看一个简单的 **缓存拦截器** 示例，它从缓存中返回响应。在真实场景中，我们会考虑 TTL、缓存失效、缓存大小等因素，但这超出本次讨论范围。这里提供一个基本示例以展示主要概念。

`cache.interceptor.ts`

```ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, of } from 'rxjs';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
```

```js
import { Injectable } from '@nestjs/common';
import { of } from 'rxjs';

@Injectable()
export class CacheInterceptor {
  intercept(context, next) {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
```

我们的 `CacheInterceptor` 使用了硬编码的 `isCached` 变量以及硬编码响应 `[]`。关键点是：我们在这里返回了一个由 RxJS `of()` 操作符创建的新流，因此路由处理器 **不会被调用**。当有人调用使用 `CacheInterceptor` 的端点时，会立即返回响应（硬编码的空数组）。要创建通用解决方案，可以利用 `Reflector` 并创建自定义装饰器。`Reflector` 在 [守卫](https://docs.nestjs.com/guards) 章节中有详细描述。

## More operators  更多操作符

使用 RxJS 操作符操作流的能力为我们提供了很多可能性。再来看一个常见用例：处理路由请求的 **超时**。当端点在一段时间后仍无返回时，你希望以错误响应终止请求。如下构造可以实现这一点：

`timeout.interceptor.ts`

```ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  RequestTimeoutException,
} from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  }
}
```

```js
import { Injectable, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor {
  intercept(context, next) {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  }
}
```

在 5 秒后，请求处理将被取消。你还可以在抛出 `RequestTimeoutException` 之前添加自定义逻辑（例如释放资源）。
