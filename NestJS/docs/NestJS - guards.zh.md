# Guards（NestJS 守卫）

来源：[NestJS Guards 文档](https://docs.nestjs.com/guards)

## Guards  守卫

守卫是一个使用 `@Injectable()` 装饰器标注并实现 `CanActivate` 接口的类。

守卫有 **单一职责**：根据运行时的一些条件（如权限、角色、ACL 等）决定当前请求是否由路由处理器处理。这通常被称为 **授权**（authorization）。授权（以及与之通常协作的 **认证** authentication）在传统 Express 应用中通常由[中间件](https://docs.nestjs.com/middleware)处理。中间件是认证的好选择，因为如令牌验证、向 `request` 对象附加属性等并不与具体路由上下文强相关。

但中间件本质上很“笨”。它不知道调用 `next()` 之后会执行哪个处理器。相对地，**守卫** 可以访问 `ExecutionContext` 实例，因此知道接下来要执行什么。它们被设计得与异常过滤器、管道、拦截器类似，能在请求/响应周期中的恰当位置插入处理逻辑，并以声明式方式完成，从而让代码更 DRY、更具声明性。

> **提示**
> 守卫在所有中间件之后执行，但在任何拦截器或管道之前执行。

### Authorization guard  授权守卫

如前所述，**授权** 是守卫的重要用例，因为某些路由应仅在调用者（通常是特定认证用户）具备足够权限时才可访问。我们将构建的 `AuthGuard` 假设用户已认证（因此请求头中带有 token）。它会提取并验证 token，并据此判断请求是否可以继续。

`auth.guard.ts`

```ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
```

```js
import { Injectable } from '@nestjs/common';

@Injectable()
export class AuthGuard {
  async canActivate(context) {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
```

> **提示**
> 若你在寻找真实世界的认证实现示例，请看[此章节](https://docs.nestjs.com/security/authentication)。更复杂的授权示例请看[此页面](https://docs.nestjs.com/security/authorization)。

`validateRequest()` 的逻辑可以简单也可以复杂。该示例的重点是展示守卫在请求/响应周期中的位置。

每个守卫必须实现 `canActivate()` 函数。该函数应返回布尔值，指示当前请求是否允许。它可以同步或异步返回（`Promise` 或 `Observable`）。Nest 使用返回值来决定下一步：

- 返回 `true`：请求被处理
- 返回 `false`：Nest 拒绝该请求

## Official enterprise support  官方企业支持

- 提供技术指导
- 深度代码评审
- 团队成员辅导
- 最佳实践建议

[了解更多](https://enterprise.nestjs.com)

### Execution context  执行上下文

`canActivate()` 函数接收一个参数 `ExecutionContext` 实例。`ExecutionContext` 继承自 `ArgumentsHost`。我们在异常过滤器章节中见过 `ArgumentsHost`。在上面的示例中，我们只是使用了 `ArgumentsHost` 上的同样辅助方法来获取 `Request` 对象。可以回顾[异常过滤器](https://docs.nestjs.com/exception-filters#arguments-host)章节的 **Arguments host** 部分了解更多。

通过扩展 `ArgumentsHost`，`ExecutionContext` 还添加了若干新的辅助方法，用于提供当前执行过程的更多细节。这些细节有助于构建通用守卫，使其能在广泛的控制器、方法和执行上下文中工作。更多 `ExecutionContext` 细节请看[这里](https://docs.nestjs.com/fundamentals/execution-context)。

### Role-based authentication  基于角色的认证

我们来构建一个更实用的守卫，它只允许具有特定角色的用户访问。先从基础模板开始，本节中它允许所有请求通过：

`roles.guard.ts`

```ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean | Promise<boolean> | Observable<boolean> {
    return true;
  }
}
```

```js
import { Injectable } from '@nestjs/common';

@Injectable()
export class RolesGuard {
  canActivate(context) {
    return true;
  }
}
```

### Binding guards  绑定守卫

与管道和异常过滤器类似，守卫可以是 **控制器级**、方法级或全局级。下面通过 `@UseGuards()` 装饰器设置控制器级守卫。该装饰器可接收单个参数或逗号分隔的多个参数，便于一次性应用合适的守卫集合。

```
@Controller('cats')
@UseGuards(RolesGuard)
export class CatsController {}
```

> **提示**
> `@UseGuards()` 装饰器来自 `@nestjs/common` 包。

上面我们传入 `RolesGuard` 类（而不是实例），由框架负责实例化并支持依赖注入。与管道和异常过滤器一样，也可以传入就地实例：

```
@Controller('cats')
@UseGuards(new RolesGuard())
export class CatsController {}
```

以上构造会将守卫应用于该控制器中声明的每个处理器。若仅想应用到单个方法，则在 **方法级** 使用 `@UseGuards()`。

设置全局守卫需使用 Nest 应用实例的 `useGlobalGuards()` 方法：

```
const app = await NestFactory.create(AppModule);
app.useGlobalGuards(new RolesGuard());
```

> **注意**
> 在混合应用中，`useGlobalGuards()` 默认不会为网关和微服务设置守卫（详见[混合应用](https://docs.nestjs.com/faq/hybrid-application)）。在“标准”（非混合）微服务应用中，`useGlobalGuards()` 会全局挂载守卫。

全局守卫作用于整个应用，对所有控制器和路由处理器生效。在依赖注入方面，从任何模块之外注册的全局守卫（如上使用 `useGlobalGuards()`）无法注入依赖，因为这发生在模块上下文之外。为了解决这一问题，你可以直接从任一模块设置守卫，使用如下构造：

`app.module.ts`

```
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: RolesGuard,
    },
  ],
})
export class AppModule {}
```

> **提示**
> 使用此方式进行守卫依赖注入时，请注意无论在哪个模块中使用该构造，守卫实际上都是全局的。应放在守卫（上例中的 `RolesGuard`）定义所在的模块中。另外，`useClass` 并非注册自定义提供者的唯一方式，更多请看[这里](https://docs.nestjs.com/fundamentals/custom-providers)。

### Setting roles per handler  为处理器设置角色

当前 `RolesGuard` 能工作，但还不够智能。它还没利用守卫最重要的特性——[执行上下文](https://docs.nestjs.com/fundamentals/execution-context)。它尚不知道角色，以及每个处理器允许的角色。比如 `CatsController` 不同路由可能有不同权限策略：一些仅 admin 可访问，一些对所有人开放。如何灵活且可复用地将角色与路由匹配？

这就是 **自定义元数据** 的用武之地（详见[这里](https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata)）。Nest 支持通过 `Reflector.createDecorator` 静态方法创建装饰器，或使用内置的 `@SetMetadata()` 装饰器，把自定义 **元数据** 附加到路由处理器上。

例如，使用 `Reflector.createDecorator` 创建 `@Roles()` 装饰器，将元数据附加到处理器上。`Reflector` 由框架内置并从 `@nestjs/core` 导出。

`roles.decorator.ts`

```
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

这里的 `Roles` 装饰器是一个接收 `string[]` 参数的函数。

使用该装饰器很简单：

`cats.controller.ts`

```
@Post()
@Roles(['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

```js
@Post()
@Roles(['admin'])
@Bind(Body())
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

这里我们将 `Roles` 元数据附加到 `create()` 方法上，表示只有 `admin` 角色的用户可访问该路由。

另外，也可以使用内置的 `@SetMetadata()` 装饰器替代 `Reflector.createDecorator`。更多请看[这里](https://docs.nestjs.com/fundamentals/execution-context#low-level-approach)。

### Putting it all together  综合应用

现在把 `RolesGuard` 与角色结合起来。目前它始终返回 `true`，允许所有请求。我们要根据 **当前用户的角色** 与当前路由所需角色匹配结果，决定返回值。要访问路由的角色（自定义元数据），我们再次使用 `Reflector`：

`roles.guard.ts`

```
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from './roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get(Roles, context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
```

```js
import { Injectable, Dependencies } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from './roles.decorator';

@Injectable()
@Dependencies(Reflector)
export class RolesGuard {
  constructor(reflector) {
    this.reflector = reflector;
  }

  canActivate(context) {
    const roles = this.reflector.get(Roles, context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
```

> **提示**
> 在 Node.js 世界中，通常会把已授权用户附加到 `request` 对象上。因此上面的示例假设 `request.user` 包含用户实例与允许的角色。在你的应用中，你可能会在自定义 **认证守卫**（或中间件）中完成该关联。更多请看[此章节](https://docs.nestjs.com/security/authentication)。

> **警告**
> `matchRoles()` 内部逻辑可以简单也可以复杂。该示例的重点是展示守卫在请求/响应周期中的位置。

关于在上下文中使用 `Reflector` 的更多信息，请参阅 **Execution context** 章节中的[反射与元数据](https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata)部分。

当用户权限不足访问端点时，Nest 会自动返回如下响应：

```
{"statusCode":403,"message":"Forbidden resource","error":"Forbidden"}
```

注意，当守卫返回 `false` 时，框架会抛出 `ForbiddenException`。如果你想返回不同的错误响应，应抛出你自己的特定异常。例如：

```
throw new UnauthorizedException();
```

守卫抛出的任何异常都会由[异常层](https://docs.nestjs.com/exception-filters)处理（全局异常过滤器和当前上下文应用的异常过滤器）。

> **提示**
> 若需真实世界的授权实现示例，请看[此章节](https://docs.nestjs.com/security/authorization)。
